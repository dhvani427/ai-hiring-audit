# AI vs. Human Hiring Scores: Agreement and Disagreement

```js
const data = FileAttachment("data/ai_hiring_audit_dataset.csv").csv({typed: true});
```

```js
function legend(container, color) {
  const categories = ["Shortlisted", "Rejected"];

  const paddingTop = 45;
  const entrySpacing = 20;
  const titleOffset = 15;
  const symbolRadius = 5;
  const labelOffset = 8;

  container.append("text")
    .style("font", "10pt sans-serif")
    .style("font-weight", "bold")
    .attr("y", paddingTop)
    .text("Final Decision");

  const entries = container.selectAll("g")
    .data(categories)
    .join("g")
      .attr(
        "transform",
        (d, i) =>
          `translate(0, ${(i * entrySpacing) + paddingTop + titleOffset})`
      );

  entries.append("circle")
    .attr("r", symbolRadius)
    .attr("fill", d => color(d));

  entries.append("text")
    .attr("x", symbolRadius + labelOffset)
    .attr("dy", "0.3em")
    .style("font", "10pt sans-serif")
    .text(d => d);

  return container.node();
}
```

```js
function attachTooltip(target, tooltip, fn) {
  target
    .on("mouseenter", function(event, datum) {
      d3.select(this)
        .attr("stroke", "black")
        .attr("stroke-width", 1.5);

      const [x, y] = d3.pointer(event);

      tooltip
        .style("visibility", "visible")
        .style("top", `${y + 10}px`)
        .style("left", `${x + 10}px`)
        .html(fn(datum));
    })
    .on("mouseout", function(event, datum) {
      d3.select(this)
        .attr("stroke", null);

      tooltip
        .style("visibility", "hidden");
    });

  return target;
}
```

```js
const selectedJob = view( Inputs.select(
    ["All", ...new Set(data.map(d => d.Job_Category))],
    {label: "Job Category"}
));
```

```js
function experienceRangeSlider(min, max, startingMin = min, startingMax = max) {
  const width = 220;
  const height = 58;
  const margin = {top: 18, right: 8, bottom: 26, left: 8};
  const trackY = margin.top + 6;

  const container = htl.html`
    <style>
      .experience-range-slider .selection {
        fill: #4a7fc1;
        fill-opacity: 0.2;
        stroke: #4a7fc1;
        stroke-width: 1.5;
        cursor: grab;
      }
      .experience-range-slider .handle {
        fill: #fff;
        stroke: #4a7fc1;
        stroke-width: 2;
        cursor: ew-resize;
        rx: 2;
      }
      .experience-range-slider .overlay {
        cursor: crosshair;
      }
    </style>
    <div class="experience-range-slider"></div>
  `;

  const svg = d3.create("svg")
    .attr("width", width)
    .attr("height", height)
    .attr("viewBox", `0 0 ${width} ${height}`);

  const x = d3.scaleLinear()
    .domain([min, max])
    .range([margin.left, width - margin.right]);

  const rangeLabel = svg.append("text")
    .attr("x", width / 2)
    .attr("y", 12)
    .attr("text-anchor", "middle")
    .style("font", "10pt sans-serif")
    .style("fill", "#333");

  svg.append("line")
    .attr("x1", margin.left)
    .attr("x2", width - margin.right)
    .attr("y1", trackY)
    .attr("y2", trackY)
    .attr("stroke", "#d0d0d0")
    .attr("stroke-width", 6)
    .attr("stroke-linecap", "round");

  const rangeFill = svg.append("rect")
    .attr("y", trackY - 3)
    .attr("height", 6)
    .attr("fill", "#4a7fc1")
    .attr("opacity", 0.55)
    .attr("rx", 3);

  const tickCount = Math.min(max - min + 1, 10);

  svg.append("g")
    .attr("transform", `translate(0, ${height - margin.bottom + 4})`)
    .call(
      d3.axisBottom(x)
        .ticks(tickCount)
        .tickSize(4)
        .tickFormat(d => `${d}`)
    )
    .call(g => g.select(".domain").remove())
    .call(g => g.selectAll("text").style("font", "9pt sans-serif").style("fill", "#666"));

  function updateRange(range) {
    const [lo, hi] = range;
    rangeLabel.text(`${lo} – ${hi} years`);
    rangeFill
      .attr("x", x(lo))
      .attr("width", Math.max(0, x(hi) - x(lo)));
    container.value = range;
    container.dispatchEvent(new CustomEvent("input"));
  }

  const brush = d3.brushX()
    .extent([[margin.left, margin.top], [width - margin.right, trackY + 14]])
    .on("brush end", ({selection}) => {
      if (!selection) return;
      const values = selection.map(x.invert).map(v => Math.round(Math.max(min, Math.min(max, v))));
      updateRange([Math.min(values[0], values[1]), Math.max(values[0], values[1])]);
    });

  svg.append("g")
    .call(brush)
    .call(brush.move, [startingMin, startingMax].map(x));

  container.querySelector(".experience-range-slider").append(svg.node());
  updateRange([startingMin, startingMax].map(Math.round));

  return container;
}
```

```js
const experienceRangeInput = (() => {
  const slider = experienceRangeSlider(
    d3.min(data, d => d.Years_Experience),
    d3.max(data, d => d.Years_Experience)
  );
  const form = htl.html`<form class="inputs-3a86ea">
    <label>Years of Experience</label>
    <div class="inputs-3a86ea-input">${slider}</div>
  </form>`;
  slider.addEventListener("input", () => form.dispatchEvent(new CustomEvent("input")));
  Object.defineProperty(form, "value", {
    get: () => slider.value,
    set: v => { slider.value = v; }
  });
  return form;
})();

const experienceRange = view(experienceRangeInput);
```
```js
const selectedDisagreement = Mutable("All");
const setSelectedDisagreement = d => {
  selectedDisagreement.value = d;
};
```

```js
function disagreementType(d) {
  const diff = d.AI_Score - d.Human_Score;

  if (diff > 5) return "AI scored higher";
  if (diff < -5) return "Human scored higher";
  return "Similar scores";
}
```

```js
const filteredData = data.filter(d =>
  (selectedJob === "All" || d.Job_Category === selectedJob) &&
  d.Years_Experience >= experienceRange[0] &&
  d.Years_Experience <= experienceRange[1]
);
```

```js
const baseFilteredData = data.filter(d =>
  (selectedJob === "All" || d.Job_Category === selectedJob) &&
  d.Years_Experience >= experienceRange[0] &&
  d.Years_Experience <= experienceRange[1]
);
```

```js
const disagreementData = [
  {
    type: "AI scored higher",
    count: baseFilteredData.filter(d => d.AI_Score - d.Human_Score > 5).length
  },
  {
    type: "Similar scores",
    count: baseFilteredData.filter(d => Math.abs(d.AI_Score - d.Human_Score) <= 5).length
  },
  {
    type: "Human scored higher",
    count: baseFilteredData.filter(d => d.Human_Score - d.AI_Score > 5).length
  }
];
```

```js
const disagreementChart = (() => {
  const width = 420;
  const height = 500;
  const margin = {top: 60, right: 15, bottom: 60, left: 55};
  const prev = globalThis.__previousBarCounts ?? {};

  const container = htl.html`<div></div>`;
  const svg = d3.select(container).append("svg")
    .attr("width", width)
    .attr("height", height);

  const x = d3.scaleBand()
    .domain(disagreementData.map(d => d.type))
    .range([margin.left, width - margin.right])
    .padding(0.25);

  const y = d3.scaleLinear()
    .domain([0, 900])
    .range([height - margin.bottom, margin.top]);

  const t = d3.transition().duration(900).ease(d3.easeCubicInOut);

  svg.append("text")
    .attr("x", width / 2)
    .attr("y", 25)
    .attr("text-anchor", "middle")
    .style("font", "20px serif")
    .style("font-weight", "bold")
    .text("Score Disagreement Summary");

  svg.append("g")
    .attr("transform", `translate(0, ${height - margin.bottom})`)
    .call(d3.axisBottom(x))
    .selectAll("text")
    .style("font", "9pt sans-serif");

  svg.append("g")
    .attr("class", "y-axis")
    .attr("transform", `translate(${margin.left}, 0)`)
    .call(d3.axisLeft(y));

  const bars = svg.selectAll("rect")
    .data(disagreementData, d => d.type)
    .join("rect")
    .attr("x", d => x(d.type))
    .attr("width", x.bandwidth())
    .attr("fill", "#444")
    .attr("opacity", d => selectedDisagreement === "All" || selectedDisagreement === d.type ? 0.85 : 0.3)
    .attr("cursor", "pointer")
    .attr("y", d => y(prev[d.type] ?? 0))
    .attr("height", d => Math.max(0, y(0) - y(prev[d.type] ?? 0)))
    .on("click", function(event, d) {
      const nextValue = selectedDisagreement === d.type ? "All" : d.type;
      setSelectedDisagreement(nextValue);
    });

  const labels = svg.selectAll("text.count")
    .data(disagreementData, d => d.type)
    .join("text")
    .attr("class", "count")
    .attr("x", d => x(d.type) + x.bandwidth() / 2)
    .attr("text-anchor", "middle")
    .style("font", "9pt sans-serif")
    .attr("y", d => y(prev[d.type] ?? 0) - 6)
    .text(d => d.count);

  svg.append("text")
    .attr("transform", "rotate(-90)")
    .attr("x", -height / 2)
    .attr("y", 20)
    .attr("text-anchor", "middle")
    .style("font", "10pt sans-serif")
    .text("Number of candidates");

  requestAnimationFrame(() => {
    bars.transition(t)
      .attr("y", d => y(d.count))
      .attr("height", d => Math.max(0, y(0) - y(d.count)));

    labels.transition(t)
      .attr("y", d => y(d.count) - 6);
  });

  globalThis.__previousBarCounts = Object.fromEntries(
    disagreementData.map(d => [d.type, d.count])
  );

  return container;
})();
```

```js
const scatterplot = (() => {
  const color = d3.scaleOrdinal()
  .domain(["Shortlisted", "Rejected"])
  .range(["#0072B2", "#E69F00"]); // Okabe-Ito blue / vermillion — darker, colorblind-safe

  const width = 900;
  const height = 500;
  const margin = {top: 60, right: 30, bottom: 60, left: 70};
  const plotWidth = 750;

  // Create SVG
  const svg = d3.create("svg")
    .attr("width", width)
    .attr("height", height);

  svg.append("text")
    .attr("x", plotWidth / 2)
    .attr("y", 25)
    .attr("text-anchor", "middle")
    .style("font", "20px serif")
    .style("font-weight", "bold")
    .text("Where Do AI and Human Hiring Scores Agree or Disagree?");

  const container = htl.html`
    <style>
      .scatterplot-container {
        position: relative;
        display: inline-block;
      }
      #tooltip {
        font: 10pt sans-serif;
        background-color: white;
        border: 1pt solid grey;
        padding: 5px;
        box-shadow: 3px 3px 3px darkgrey;
        max-width: 40ch;
        z-index: 1000;
        visibility: hidden;
        position: fixed;
        pointer-events: none;
      }
    </style>
    <div class="scatterplot-container">
      <div id="tooltip"></div>
      ${svg.node()}
    </div>
  `;

  const x = d3.scaleLinear()
    .domain(d3.extent(data, d => d.Human_Score))
    .range([margin.left, plotWidth - margin.right])
    .nice();

  const y = d3.scaleLinear()
    .domain(d3.extent(data, d => d.AI_Score))
    .range([height - margin.bottom, margin.top])
    .nice();

  svg.append('g')
    .attr("transform", `translate(0, ${height - margin.bottom})`)
    .call(d3.axisBottom(x));

  svg.append('g')
    .attr("transform", `translate(${margin.left}, 0)`)
    .call(d3.axisLeft(y));

  svg.append("text")
    .attr("x", plotWidth / 2)
    .attr("y", height - 15)
    .attr("text-anchor", "middle")
    .style("font", "10pt sans-serif")
    .text("Human Score");

  svg.append("text")
    .attr("transform", "rotate(-90)")
    .attr("x", -height / 2)
    .attr("y", 20)
    .attr("text-anchor", "middle")
    .style("font", "10pt sans-serif")
    .text("AI Score");

  // Reference line: AI Score = Human Score
  svg.append("line")
    .attr("x1", x(0))
    .attr("y1", y(0))
    .attr("x2", x(100))
    .attr("y2", y(100))
    .attr("stroke", "black")
    .attr("stroke-width", 3)
    .attr("stroke-dasharray", "8 6")
    .attr("opacity", 0.7);

  svg.append("text")
    .attr("x", x(90))
    .attr("y", y(102))
    .style("font", "9pt sans-serif")
    .style("fill", "black")
    .text("AI Score = Human Score");
      
  const tooltip = d3.select(container).select("#tooltip");
  const plotLeft = margin.left;
  const plotTop = margin.top;
  const plotRight = plotWidth - margin.right;
  const plotBottom = height - margin.bottom;

  const candidates = svg.selectAll("g.candidate")
    .data(filteredData, d => d.Candidate_ID)
    .join("g")
      .attr("class", "candidate")
      .attr("transform", d => `translate(${x(d.Human_Score)}, ${y(d.AI_Score)})`);

  candidates.selectAll("circle.visible")
    .data(d => [d])
    .join("circle")
      .attr("class", "visible")
      .attr("r", 4)
      .attr("pointer-events", "none")
      .attr("opacity", d =>
        selectedDisagreement === "All" || disagreementType(d) === selectedDisagreement
          ? 0.65
          : 0.15
      )
      .attr("fill", d => color(d.Final_Decision === 1 ? "Shortlisted" : "Rejected"));

  const delaunay = d3.Delaunay.from(
    filteredData,
    d => x(d.Human_Score),
    d => y(d.AI_Score)
  );
  const voronoi = delaunay.voronoi([plotLeft, plotTop, plotRight, plotBottom]);

  function candidateTooltip(d) {
    return `
      <strong>Candidate ${d.Candidate_ID}</strong><br>
      Job: ${d.Job_Category}<br>
      Education: ${d.Education_Level}<br>
      Experience: ${d.Years_Experience} years<br>
      Human Score: ${d.Human_Score}<br>
      AI Score: ${d.AI_Score}<br>
      Final Decision: ${d.Final_Decision === 1 ? "Shortlisted" : "Rejected"}
    `;
  }

  svg.append("g")
    .attr("class", "voronoi")
    .selectAll("path")
    .data(filteredData, d => d.Candidate_ID)
    .join("path")
      .attr("d", (_, i) => voronoi.renderCell(i))
      .attr("fill", "transparent")
      .attr("stroke", "none")
      .attr("pointer-events", "all")
      .attr("cursor", "pointer")
      .on("mousemove", function(event, d) {
        candidates.select("circle.visible")
          .attr("stroke", c => c.Candidate_ID === d.Candidate_ID ? "black" : null)
          .attr("stroke-width", c => c.Candidate_ID === d.Candidate_ID ? 1.5 : null);

        tooltip
          .style("visibility", "visible")
          .style("top", `${event.clientY + 12}px`)
          .style("left", `${event.clientX + 12}px`)
          .html(candidateTooltip(d));
      })
      .on("mouseleave", () => {
        candidates.select("circle.visible")
          .attr("stroke", null);

        tooltip
          .style("visibility", "hidden");
      });
  
  const legendGroup = svg.append("g")
  .attr("transform", `translate(${plotWidth + 20}, 40)`);
  
  legend(legendGroup, color);

  return container;
}
)();
```

```js
html`
<div>
  <div style="
    display:flex;
    gap:24px;
    align-items:flex-start;
  ">
    <div>${scatterplot}</div>
    <div>${disagreementChart}</div>
  </div>

  <div style="
    font: 10pt sans-serif;
    margin-top: 8px;
    color: #555;
    line-height: 1.3;
    text-align: center;
  ">
    <div>Showing candidates with ${experienceRange[0]} to ${experienceRange[1]} years of experience or less.</div>
    <div>Showing ${filteredData.length} of ${data.length} candidates.</div>
    <div>Highlighted group: ${selectedDisagreement}</div>
  </div>

  <div style="
    display:flex;
    gap:24px;
    margin-top:10px;
  ">
    <div style="
      width:900px;
      font: 10pt sans-serif;
      line-height:1.5;
    ">
      This scatterplot compares AI and human scores for each candidate. Points above the dashed line were scored higher by AI, while points below were scored higher by humans.
    </div>

    <div style="
      width:420px;
      font: 10pt sans-serif;
      line-height:1.5;
    ">
      This chart summarizes disagreement types. Scores within 5 points are counted as similar. Click a bar to highlight that group in the scatterplot.
    </div>
  </div>
</div>
`
```
## Design Rationale

I chose a scatterplot because the main goal is to compare AI and human hiring scores for each candidate. The dashed diagonal line shows perfect agreement between the two scores. Points above the line were scored higher by AI, while points below the line were scored higher by humans.

Color shows the final hiring decision, with blue for shortlisted candidates and orange for rejected candidates. I added a job category dropdown and years-of-experience range slider so viewers can explore how score agreement changes across different groups. The slider is adapted from a D3 brush example on Observable (see References). Tooltips provide candidate details without cluttering the chart.

I also included a summary bar chart that counts three disagreement types: AI scored higher, human scored higher, and similar scores.  I defined similar scores as cases where the AI and human scores differ by 5 points or less. The bars are clickable, so selecting a bar highlights that disagreement group in the scatterplot while fading the other points.

I considered using only a bar chart or summary table, but those would hide individual candidate-level differences. The final design combines an overview of disagreement patterns with detailed candidate-level exploration, while the linked highlighting interaction keeps the full scatterplot visible for context.

## Data Source

AI-Assisted Hiring Fairness and Bias Audit Dataset by Zulqarnain Haider on Kaggle. The dataset contains 1,500 simulated candidate records and is intended for analyzing AI/human hiring alignment, disagreement, and bias patterns.

Dataset link: [https://www.kaggle.com/datasets/zulqarnain11/zzzzzzzzzzzzzzzz](https://www.kaggle.com/datasets/zulqarnain11/zzzzzzzzzzzzzzzz)

## References

https://observablehq.com/@sarah37/snapping-range-slider-with-d3-brush

## Project Repository

[View the GitHub repository](https://github.com/dhvani427/ai-hiring-audit)