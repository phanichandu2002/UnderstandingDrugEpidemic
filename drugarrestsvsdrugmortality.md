

# phani drug mortality vs drug usage
```js
const drugMortalityData = await FileAttachment("Drug mortality.csv").csv({typed: true});
const drugarrestsData=await FileAttachment("DrugArrests.csv").csv({typed: true});
const drugMortalityLong = drugMortalityData.flatMap(row => {
  return Object.keys(row)
    .filter(key => key !== 'County')
    .map(year => ({
      County: row.County,
      Year: parseInt(year),
      DrugMortality: parseFloat(row[year])
    }));
});

const drugArrestsLong = drugarrestsData.flatMap(row => {
  return Object.keys(row)
    .filter(key => key !== 'County')
    .map(year => ({
      County: row.County,
      Year: parseInt(year),
      DrugArrests: parseFloat(row[year])
    }));
});

const mergedData = drugMortalityLong.map(dm => {
  const arrestData = drugArrestsLong.find(da => 
    da.County === dm.County && da.Year === dm.Year
  );
  return arrestData ? { ...dm, DrugArrests: arrestData.DrugArrests } : null;
}).filter(d => d);

display(mergedData);
```
```js
function renderScatterPlot(data, selectedYear) {
  const filteredData = data.filter(d => d.Year === selectedYear);

  return Plot.plot({
    title: `Drug Mortality vs. Drug Arrests (${selectedYear})`,
    width: 600,
    height: 300,
    x: { label: "Drug Arrests (per 100k)", grid: true },
    y: { label: "Drug Mortality (per 100k)", grid: true },
    marks: [
      Plot.dot(filteredData, {
        x: "DrugArrests",
        y: "DrugMortality",
        fill: "County",
        tip: true,
      }),
    ],
  });
}

// Function to update the scatter plot when the slider changes
function updateScatterPlot(selectedYear) {
  d3.select("#scatterplot-container").html(""); // Clear the container
  const scatterPlot = renderScatterPlot(mergedData, selectedYear);
  d3.select("#scatterplot-container").append(() => scatterPlot);
}

// Attach scatter plot update to the year slider
drugMortalityYearSlider.addEventListener("input", () => {
  const selectedYear = parseInt(drugMortalityYearSlider.value);
  updateScatterPlot(selectedYear);
});

// Initial render for the default year
updateScatterPlot(drugMortalityYearSlider.value);

```
```js
function renderDrugMortalityBarChart(data, selectedYear) {
  const filteredData = data.filter(d => d.Year === selectedYear);

  return Plot.plot({
    title: `Drug Mortality by County (${selectedYear})`,
    width: 600,
    height: 300,
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
}

// Create a year slider
const drugMortalityYears = Array.from(new Set(mergedData.map(d => d.Year))).sort();
const drugMortalityYearSlider = Inputs.range([Math.min(...drugMortalityYears), Math.max(...drugMortalityYears)], {
  label: "Select Year for Drug Mortality Chart",
  step: 1,
  value: drugMortalityYears[0], // Default to the first year
});

// Append the slider and render the chart
display(drugMortalityYearSlider);
function updateDrugMortalityBarChart(selectedYear) {
  d3.select("#barchart-container").html("");
  const barChart = renderDrugMortalityBarChart(mergedData, selectedYear);
  d3.select("#barchart-container").append(() => barChart);
}
updateDrugMortalityBarChart(drugMortalityYearSlider.value);

drugMortalityYearSlider.addEventListener("input", () => {
  const selectedYear = parseInt(drugMortalityYearSlider.value);
  updateDrugMortalityBarChart(selectedYear);
});
```
```js
function renderDrugMortalityLineGraph(data) {
  return Plot.plot({
    title: "Drug Mortality Over Time by County",
    width: 600,
    height: 300,
    x: { label: "Year", grid: true },
    y: { label: "Drug Mortality (per 100k)", grid: true },
    color: { legend: true },
    marks: [
      Plot.line(data, {
        x: "Year",
        y: "DrugMortality",
        stroke: "County", // Unique color for each county
        strokeWidth: 1.0,
        tip: true,
      }),
    ],
  });
}

// Render the line graph
d3.select("#linegraph-container").html(""); // Clear the container
d3.select("#linegraph-container").append(() => renderDrugMortalityLineGraph(mergedData));
```

<div id="scatterplot-container" style="border: 1px solid #ddd; padding: 10px;"></div>
<div id="barchart-container" style="border: 1px solid #ddd; padding: 10px;"></div>
<div id="linegraph-container" style="border: 1px solid #ddd; padding: 10px;"></div>
