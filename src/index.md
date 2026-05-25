# AI vs. Human Hiring Scores: Agreement and Disagreement

```js
const data = FileAttachment("data/ai_hiring_audit_dataset.csv").csv({typed: true});
```

```js
function legend(container, color) {
  const categories = ["Shortlisted", "Rejected"];

  const paddingTop = 15;
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
const selectedJobInput = Inputs.select(
    ["All", ...new Set(data.map(d => d.Job_Category))]
  );

const selectedJob = Generators.input(selectedJobInput);
```

```js
const selectedYearsInput =
  Inputs.range(
    [
      d3.min(data, d => d.Years_Experience),
      d3.max(data, d => d.Years_Experience)
    ],
    {
      step: 1,
      value: d3.max(data, d => d.Years_Experience),
      label: "Max Experience"
    }
  );

const selectedYears = Generators.input(selectedYearsInput);
```

```js
const filteredData = data.filter(d =>
  (selectedJob === "All" || d.Job_Category === selectedJob) &&
  d.Years_Experience <= selectedYears
);
```

```js
const scatterplot = (() => {
  const color = d3.scaleOrdinal()
  .domain(["Shortlisted", "Rejected"])
  .range(["green", "red"]);
  
  const width = 760;
  const height = 500;
  const margin = {top: 40, right: 30, bottom: 60, left: 70};
  const plotWidth = 610;

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
      #tooltip {
        font: 10pt sans-serif;
        background-color: white;
        border: 1pt solid grey;
        padding: 5px;
        box-shadow: 3px 3px 3px darkgrey;
        max-width: 40ch;
        z-index: 1;
        visibility: hidden; 
        position: absolute;
      }
    </style>
    <div id="tooltip"></div>
    ${svg.node()}
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
    .attr("x", width / 2)
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
    .attr("x", x(66))
    .attr("y", y(94))
    .style("font", "9pt sans-serif")
    .style("fill", "black")
    .text("AI Score = Human Score");
      
  svg.selectAll("circle")
    .data(filteredData)
    .join("circle")
      .attr("cx", d => x(d.Human_Score))
      .attr("cy", d => y(d.AI_Score))
      .attr("r", 3)
      .attr("fill", "steelblue")
      .attr("opacity", 0.6)
      .attr("fill", d =>color(d.Final_Decision === 1 ? "Shortlisted" : "Rejected"))

  const tooltip = d3.select(container).select("#tooltip");

  svg.selectAll("circle.candidate")
    .data(filteredData)
    .join("circle")
      .attr("class", "candidate")
      .attr("cx", d => x(d.Human_Score))
      .attr("cy", d => y(d.AI_Score))
      .attr("r", 3)
      .attr("opacity", 0.6)
      .attr("fill", d => color(d.Final_Decision === 1 ? "Shortlisted" : "Rejected"))
      .call(g => attachTooltip(g, tooltip, d => `
        <strong>Candidate ${d.Candidate_ID}</strong><br>
        Job: ${d.Job_Category}<br>
        Education: ${d.Education_Level}<br>
        Experience: ${d.Years_Experience} years<br>
        Human Score: ${d.Human_Score}<br>
        AI Score: ${d.AI_Score}<br>
        Final Decision: ${d.Final_Decision === 1 ? "Shortlisted" : "Rejected"}
      `));
  
  const legendGroup = svg.append("g")
  .attr("transform", `translate(${plotWidth + 20}, 40)`);
  
  legend(legendGroup, color);

  return container;
}
)();
```

```js
html`
<div style="
  display:grid;
  grid-template-columns: 760px 260px;
  column-gap:24px;
  align-items:start;
">

  <div>
    ${scatterplot}

    <div style="
      width:610px;
      font: 10pt sans-serif;
      line-height:1.5;
      margin-top:10px;
    ">
      This scatterplot compares each candidate’s human score and AI score.
      The dashed diagonal line represents equal AI and human scores.
      Points above the dashed line were scored higher by AI than by humans,
      while points below the line were scored higher by humans. The job
      category and years of experience filters allow viewers to compare
      whether these patterns change across roles and experience levels.
    </div>
  </div>

  <div style="
    margin-top:42px;
    width:260px;
  ">

    <div style="
      font: 10pt sans-serif;
      font-weight: bold;
      margin-bottom: 12px;
    ">
      Job Category
    </div>

    <div style="width:260px;">
      ${selectedJobInput}
    </div>

    <div style="
      font: 10pt sans-serif;
      font-weight: bold;
      margin-top: 24px;
      margin-bottom: 12px;
    ">
      Years of Experience
    </div>

    <div style="width:260px;">
      ${selectedYearsInput}
    </div>

    <div style="
      font: 9pt sans-serif;
      margin-top: 8px;
      color: #555;
      width:260px;
      line-height:1.3;
      white-space: normal;
    ">
      Showing candidates with ${selectedYears} years of experience or less.
    </div>

    <div style="
      font: 9pt sans-serif;
      margin-top: 10px;
      color: #555;
    ">
      Showing ${filteredData.length} of ${data.length} candidates.
    </div>

  </div>

</div>
`
```

## Design Rationale

I chose a scatterplot because the main goal is to compare two numerical values: the human hiring score and the AI hiring score. Each point represents one candidate, which makes it easy to see the overall relationship between the two scoring systems as well as individual cases where they disagree.

The x-axis shows the human score, and the y-axis shows the AI score. I added the dashed diagonal line to represent perfect agreement between the two scores. Points near the line show candidates where AI and humans scored similarly. Points above the line were scored higher by AI, while points below the line were scored higher by humans.

Color shows the final hiring decision. Green represents shortlisted candidates, and red represents rejected candidates. This makes it easier to see how the score patterns relate to actual outcomes. I considered using a bar chart or summary averages, but those would hide individual candidates and make disagreements harder to notice.

I added a job category dropdown so viewers can compare patterns across different roles. I also added a years-of-experience slider because experience is an important factor in hiring, and the slider makes it easy to filter candidates by experience level. The tooltip gives extra details about each candidate without cluttering the chart.

Overall, this design helps viewers quickly understand where AI and human scores agree, where they differ, and how those differences connect to hiring decisions.

## Data Source

AI-Assisted Hiring Fairness and Bias Audit Dataset by Zulqarnain Haider on Kaggle. The dataset contains 1,500 simulated candidate records and is intended for analyzing AI/human hiring alignment, disagreement, and bias patterns.

Dataset link: [https://www.kaggle.com/datasets/zulqarnain11/zzzzzzzzzzzzzzzz](https://www.kaggle.com/datasets/zulqarnain11/zzzzzzzzzzzzzzzz)

## Project Repository

[View the GitHub repository](https://github.com/dhvani427/ai-hiring-audit)