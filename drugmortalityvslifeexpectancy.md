---
theme: dashboard
title: Drug Mortality vs. Life Expectancy Dashboard
toc: false
---
```js
const drugMortalityData = await FileAttachment("Drug mortality.csv").csv({typed: true});
const lifeExpectancyData = await FileAttachment("Life Expectancy.csv").csv({typed: true});
```
```js

const lifeExpectancyLong = lifeExpectancyData.flatMap(row => {
  return Object.keys(row)
    .filter(key => key !== 'County')
    .map(year => ({
      County: row.County,
      Year: parseInt(year),
      LifeExpectancy: parseFloat(row[year])
    }));
});
const drugMortalityLong = drugMortalityData.flatMap(row => {
  return Object.keys(row)
    .filter(key => key !== 'County')
    .map(year => ({
      County: row.County,
      Year: parseInt(year),
      DrugMortality: parseFloat(row[year])
    }));
});
const mergedData = lifeExpectancyLong.map(le => {
  const drugData = drugMortalityLong.find(dm => 
    dm.County === le.County && dm.Year === le.Year
  );
  return drugData ? { ...le, DrugMortality: drugData.DrugMortality } : null;
}).filter(d => d);


display(mergedData);
```
```js
const colorScale = Plot.scale({
  color: {
    type: "sequential",
    domain: [0, d3.max(mergedData, d => d.DrugMortality)],
    range: ["lightblue", "darkred"]
  }
});
```
```js
function mortalityVsLifeExpectancy(data, {width} = {}) {
  return Plot.plot({
    title: "Drug Mortality vs. Life Expectancy",
    width,
    height: 200,
    x: {label: "Life Expectancy (Years)", grid: true},
    y: {label: "Drug Mortality (per 100k)", grid: true},
    color: {...colorScale, legend: true},
    marks: [
      Plot.dot(data, {
        x: "LifeExpectancy",
        y: "DrugMortality",
        fill: "DrugMortality",
        tip: true
      })
    ]
  });
}
```
```js
// Create the year slider
const years = Array.from(new Set(mergedData.map(d => d.Year))).sort();
const yearSlider = Inputs.range([Math.min(...years), Math.max(...years)], {
  label: "Select Year",
  step: 1,
  value: years[0], // Default to the first year
});
display(yearSlider);

// Create containers for the plots
display(html`<div id="dashboard" style="display: grid; grid-template-columns: repeat(2, 1fr); gap: 20px;"></div>`);

// Append individual plot containers
d3.select("#dashboard")
  .html(`
    <div id="scatterplot" style="border: 1px solid #ddd; padding: 10px;"></div>
    <div id="barchart" style="border: 1px solid #ddd; padding: 10px;"></div>
    <div id="heatmap" style="border: 1px solid #ddd; padding: 10px;"></div>
    <div id="linegraph" style="border: 1px solid #ddd; padding: 10px;"></div>
  `);

// Function to update all plots dynamically
function updateDashboard(selectedYear) {
  const filteredData = mergedData.filter(d => d.Year === selectedYear);

  // Update Scatterplot
  d3.select("#scatterplot").html("");
  const scatterplot = Plot.plot({
    title: `Drug Mortality vs. Life Expectancy (${selectedYear})`,
    width: 400,
    height: 200,
    x: { label: "Life Expectancy (Years)", grid: true },
    y: { label: "Drug Mortality (per 100k)", grid: true },
    color: { type: "sequential", domain: [0, d3.max(filteredData, d => d.DrugMortality)], range: ["lightblue", "darkred"], legend: true },
    marks: [
      Plot.dot(filteredData, {
        x: "LifeExpectancy",
        y: "DrugMortality",
        fill: "DrugMortality",
        tip: true,
      }),
    ],
  });
  d3.select("#scatterplot").append(() => scatterplot);

  // Update Bar Chart
  d3.select("#barchart").html("");
  const barChart = Plot.plot({
    title: `Drug Mortality by County (${selectedYear})`,
    width: 400,
    height: 200,
    x: {
      label: "County",
      domain: filteredData.map(d => d.County),
      tickRotate: -45,
      grid: true,
    },
    y: { label: "Drug Mortality (per 100k)", grid: true },
    marks: [
      Plot.barY(filteredData, {
        x: "County",
        y: "DrugMortality",
        fill: "County",
        tip: true,
      }),
    ],
  });
  d3.select("#barchart").append(() => barChart);

  // Update Heatmap
  d3.select("#heatmap").html("");
  const heatmap = Plot.plot({
    title: `Heatmap of Drug Mortality vs. Life Expectancy (${selectedYear})`,
    width: 400,
    height: 200,
    x: { label: "Life Expectancy (Years)", grid: true, domain: d3.extent(filteredData, d => d.LifeExpectancy) },
    y: { label: "Drug Mortality (per 100k)", grid: true, domain: d3.extent(filteredData, d => d.DrugMortality) },
    color: { type: "sequential", domain: [0, d3.max(filteredData, d => d.DrugMortality)], range: ["lightblue", "darkred"], legend: true },
    marks: [
      Plot.rect(filteredData, {
        x: "LifeExpectancy",
        y: "DrugMortality",
        fill: "DrugMortality",
        inset: 0.5,
      }),
    ],
  });
  d3.select("#heatmap").append(() => heatmap);

  // Update Line Graph
  d3.select("#linegraph").html("");
  const lineGraph = Plot.plot({
    title: `Drug Mortality and Life Expectancy (${selectedYear})`,
    width: 400,
    height: 200,
    x: {
      label: "County",
      domain: filteredData.map(d => d.County),
      tickRotate: -45,
      grid: true,
    },
    y: { label: "Value", grid: true },
    marks: [
      Plot.lineY(filteredData, { x: "County", y: "LifeExpectancy", stroke: "blue", tip: true }),
      Plot.dot(filteredData, { x: "County", y: "LifeExpectancy", fill: "blue" }),
      Plot.lineY(filteredData, { x: "County", y: "DrugMortality", stroke: "red", tip: true }),
      Plot.dot(filteredData, { x: "County", y: "DrugMortality", fill: "red" }),
    ],
  });
  d3.select("#linegraph").append(() => lineGraph);
}

// Event listener to update plots when the slider changes
yearSlider.addEventListener("input", () => {
  const selectedYear = parseInt(yearSlider.value);
  updateDashboard(selectedYear);
});

// Initial render for the default year
updateDashboard(yearSlider.value);
```