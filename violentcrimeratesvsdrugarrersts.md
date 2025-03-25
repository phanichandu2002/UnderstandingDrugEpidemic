---
theme: dashboard
title: Violent Dashboard
toc: false
---

# Violent Crime rate vs drug arrests 

```js
const ViolentcrimerateData = await FileAttachment("Violentcrimerates.csv").csv({typed: true});
const DrugArrestsData = await FileAttachment("DrugArrests.csv").csv({typed: true});
```
```js

const DrugArrestsLong = DrugArrestsData.flatMap(row => {
  return Object.keys(row)
    .filter(key => key !== 'County')
    .map(year => ({
      County: row.County,
      Year: parseInt(year),
      DrugArrests: parseFloat(row[year])
    }));
});
const ViolentcrimeratesLong =ViolentcrimerateData.flatMap(row => {
  return Object.keys(row)
    .filter(key => key !== 'County')
    .map(year => ({
      County: row.County,
      Year: parseInt(year),
      Violentcrimerates: parseFloat(row[year])
    }));
});
const mergedData = ViolentcrimeratesLong.map(le => {
  const drugData = DrugArrestsLong.find(dm => 
    dm.County === le.County && dm.Year === le.Year
  );
  return drugData ? { ...le, DrugArrests: drugData.DrugArrests } : null;
}).filter(d => d);

```
```js
const colorScale = Plot.scale({
  color: {
    type: "sequential",
    domain: [0, d3.max(mergedData, d => d.Violentcrimerates)],
    range: ["violet", "darkred"]
  }
});
```
```js
function violentcrimeratesVsdrugarrests(data, {width} = {}) {
  return Plot.plot({
    title: "Violent Crime rate vs drug arrests ",
    width,
    height: 400,
    x: {label: "Violent Crime Rates", grid: true},
    y: {label: "Drug Arrests", grid: true},
    color: {...colorScale, legend: true},
    marks: [
      Plot.dot(data, {
        x: "DrugArrests",
        y: "Violentcrimerates",
        fill: "Violentcrimerates",
        tip: true
      })
    ]
  });
}
```
```js
// Get unique years for the slider
const years = Array.from(new Set(mergedData.map(d => d.Year))).sort();

// Create a slider to select the year
const yearSlider = Inputs.range([Math.min(...years), Math.max(...years)], {
  label: "Select Year",
  step: 1,
  value: years[0]
});

// Display the slider
display(yearSlider);
// Create a container to hold both visualizations side by side
display(html`<div id="visualizations" style="display: flex; gap: 20px;"></div>`);

// Function to render heatmap
function renderHeatmap(selectedYear) {
  const filteredData = mergedData.filter(d => d.Year === selectedYear);

  return Plot.plot({
    title: `Heatmap: Violent Crime Rates vs. Drug Arrests (${selectedYear})`,
    width: 400,
    height: 400,
    x: {label: "Drug Arrests", grid: true, domain: d3.extent(filteredData, d => d.DrugArrests)},
    y: {label: "Violent Crime Rates", grid: true, domain: d3.extent(filteredData, d => d.Violentcrimerates)},
    color: {
      type: "sequential",
      domain: d3.extent(filteredData, d => d.Violentcrimerates),
      range: ["lightyellow", "darkorange"], // Yellow-orange color scale
      legend: true,
    },
    marks: [
      Plot.rect(filteredData, {
        x: "DrugArrests",
        y: "Violentcrimerates",
        fill: "Violentcrimerates",
        inset: 0.5,
        stroke: "white"
      })
    ]
  });
}

// Function to render scatter plot
function renderScatterPlot(selectedYear) {
  const filteredData = mergedData.filter(d => d.Year === selectedYear);

  return Plot.plot({
    title: `Scatter Plot: Violent Crime Rates vs. Drug Arrests (${selectedYear})`,
    width: 400,
    height: 400,
    x: {label: "Drug Arrests", grid: true},
    y: {label: "Violent Crime Rates", grid: true},
    color: {
      type: "sequential",
      domain: d3.extent(filteredData, d => d.Violentcrimerates),
      range: ["lightgreen", "darkblue"], // Green-blue color scale
      legend: true
    },
    marks: [
      Plot.dot(filteredData, {
        x: "DrugArrests",
        y: "Violentcrimerates",
        fill: "Violentcrimerates",
        tip: true
      })
    ]
  });
}

// Initial render for the default year
const visualizationsDiv = d3.select("#visualizations");
visualizationsDiv.append("div").attr("id", "heatmap").append(() => renderHeatmap(yearSlider.value));
visualizationsDiv.append("div").attr("id", "scatter-plot").append(() => renderScatterPlot(yearSlider.value));

// Update visualizations when year changes
yearSlider.addEventListener("input", () => {
  d3.select("#heatmap").html("").append(() => renderHeatmap(yearSlider.value));
  d3.select("#scatter-plot").html("").append(() => renderScatterPlot(yearSlider.value));
});
```
```js
// Group data by counties
const groupedByCounty = d3.group(ViolentcrimeratesLong, d => d.County);

// Create a container to hold the line chart visualization
display(html`<div id="line-chart-container" style="width: 100%;"></div>`);

// Function to render multi-line chart for violent crime rates
function renderMultiLineChart() {
  return Plot.plot({
    title: "Violent Crime Rates Over Time by County",
    width: 600,
    height: 300,
    x: {label: "Year", grid: true},
    y: {label: "Violent Crime Rates", grid: true},
    color: {legend: true}, // Enable legend for counties
    marks: [
      Plot.line(ViolentcrimeratesLong, {
        x: "Year",
        y: "Violentcrimerates",
        stroke: "County", // Assign a unique color per county
        strokeWidth: 1.0,
        title: d => `${d.County}, ${d.Year}: ${d.Violentcrimerates}`
      })
    ]
  });
}

// Render the chart
d3.select("#line-chart-container").append(() => renderMultiLineChart());

```