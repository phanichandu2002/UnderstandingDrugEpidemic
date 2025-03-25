---
theme: dashboard
title: Correlation between UnemploymentRates and DrugArrests
toc: false
---

# UnemploymentRates vs DrugArrests

```js
const UnemployementRatesData = await FileAttachment("UnemploymentRates.csv").csv({typed: true});
const DrugArrestsData = await FileAttachment("DrugArrests.csv").csv({typed: true});
```
```js
display(Inputs.table(UnemployementRatesData));
display(Inputs.table(DrugArrestsData ));
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
const UnemployementRatesLong = UnemployementRatesData.flatMap(row => {
  return Object.keys(row)
    .filter(key => key !== 'County')
    .map(year => ({
      County: row.County,
      Year: parseInt(year),
      UnemployementRates: parseFloat(row[year])
    }));
});
const mergedData = UnemployementRatesLong.map(le => {
  const drugData = DrugArrestsLong.find(dm => 
    dm.County === le.County && dm.Year === le.Year
  );
  return drugData ? { ...le, DrugArrests: drugData.DrugArrests} : null;
}).filter(d => d);

display(Inputs.table(mergedData));
```

```js
const colorScale = Plot.scale({
  color: {
    type: "sequential",
    domain: [0, d3.max(mergedData, d => d.UnemployementRates)],
    range: ["lightblue", "darkred"]
  }
});
```
```js
function YearFilter(data) {
  // Extract unique years
  const years = Array.from(new Set(data.map(d => d.Year))).sort();

  // Create a dropdown menu for year selection
  const dropdown = Inputs.select(years, { label: "Select Year" });

  // Create a container for the scatter plot
  const scatterPlotContainer = document.createElement("div");

  // Function to update the scatter plot
  function updateScatterPlot(selectedYear) {
    // Filter data for the selected year
    const filteredData = data.filter(d => d.Year === selectedYear);

    // Clear the existing scatter plot
    scatterPlotContainer.innerHTML = "";

    // Render the updated scatter plot
    scatterPlotContainer.appendChild(
      Unemployementvsdrugarrests(filteredData)
    );
  }

  // Initial render of the scatter plot with the first year as default
  updateScatterPlot(years[0]);

  // Update the scatter plot when the dropdown value changes
  dropdown.addEventListener("input", () => {
    updateScatterPlot(dropdown.value);
  });

  // Return the dropdown and scatter plot container together
  const container = document.createElement("div");
  container.appendChild(dropdown);
  container.appendChild(scatterPlotContainer);
  return container;
}
```
```js
// Display the dropdown and the initial scatter plot
display(YearFilter(mergedData));
```
```js
function Unemployementvsdrugarrests(data, {width} = {}) {
  return Plot.plot({
    title: "Unemployemnt rates vs Drug Arrests",
    width,  
    height: 400,
    x: {label: "Drug Arrests(Raw)", grid: true},
    y: {label: "Unemployment Rate(Percent)", grid: true},
    color: {...colorScale, legend: true},
    marks: [
      Plot.dot(data, {
        x: "DrugArrests",
        y: "UnemployementRates",
        fill: "UnemployementRates",
        tip: true
      })
    ]
  });
}
```
```js
display(Unemployementvsdrugarrests(mergedData));
```
```js
let currentChart; // Reference to the current chart container

function BarChartDrugArrests(data, {width = 1000, sortOrder = null} = {}) {
    const groupedData = d3.rollups(
        data,
        v => d3.sum(v, d => d.DrugArrests),
        d => d.County
    ).map(([County, TotalDrugArrests]) => ({County, TotalDrugArrests}));

    let sortedGroupedData;
    if (sortOrder === "ascending") {
        sortedGroupedData = groupedData.sort((a, b) => a.TotalDrugArrests - b.TotalDrugArrests);
    } else if (sortOrder === "descending") {
        sortedGroupedData = groupedData.sort((a, b) => b.TotalDrugArrests - a.TotalDrugArrests);
    } else {
        // Default sort by County
        sortedGroupedData = groupedData.sort((a, b) => a.County.localeCompare(b.County));
    }

    const maxValue = d3.max(sortedGroupedData, d => d.TotalDrugArrests);
    const minValue = d3.min(sortedGroupedData, d => d.TotalDrugArrests);

    const styledData = sortedGroupedData.map(d => ({
        ...d,
        color: d.TotalDrugArrests === maxValue 
            ? "red" 
            : d.TotalDrugArrests === minValue 
                ? "green" 
                : "steelblue"
    }));

    // Plot the bar chart
    const plot = Plot.plot({
        title: `Total Drug Arrests per County`,
        width,
        height: 400,
        x: {
            label: "County",
            domain: styledData.map(d => d.County),
            tickRotate: -45
        },
        y: {label: "Total Drug Arrests"},
        marks: [
            Plot.barY(styledData, {
                x: "County",
                y: "TotalDrugArrests",
                fill: d => d.color,
                tip: d => `County: ${d.County}\nTotal Arrests: ${d.TotalDrugArrests}`
            })
        ]
    });

    // If chart already exists, remove the old chart before adding a new one
    if (currentChart) {
        currentChart.remove();
    }

    // Store the new chart reference
    currentChart = plot;
    return plot;
}

// Create buttons for toggling sorting order and unsorted state
const ascendingButton = Inputs.button("Sort Ascending");
const descendingButton = Inputs.button("Sort Descending");
const unsortedButton = Inputs.button("Reset to Unsorted");

// Add event listeners to buttons
ascendingButton.addEventListener("click", () => {
    display(BarChartDrugArrests(mergedData, {sortOrder: "ascending"}));
});

descendingButton.addEventListener("click", () => {
    display(BarChartDrugArrests(mergedData, {sortOrder: "descending"}));
});

unsortedButton.addEventListener("click", () => {
    display(BarChartDrugArrests(mergedData)); // Default to unsorted
});

// Display buttons and the initial chart
display(ascendingButton);
display(descendingButton);
display(unsortedButton);
// Display the initial chart
currentChart = BarChartDrugArrests(mergedData); // Default sort by County
display(currentChart);
```

```js
// Heatmap code for displaying unemployment rates
function generateHeatmap(data, { width = 800, height = 400 } = {}) {
  const counties = Array.from(new Set(data.map(d => d.County)));
  const years = Array.from(new Set(data.map(d => d.Year))).sort();

  const xDomain = counties;
  const yDomain = years;

  const heatmapData = data.map(d => ({
    County: d.County,
    Year: d.Year,
    Value: d.Value
  }));

  const colorScale = Plot.scale({
    color: {
      type: "linear",
      domain: [d3.min(heatmapData, d => d.Value), d3.max(heatmapData, d => d.Value)],
      range: ["lightblue", "darkred"]
    }
  });

  return Plot.plot({
    width,
    height,
    grid: true,
    x: {
      label: "Counties",
      domain: xDomain,
      tickRotate: -45
    },
    y: {
      label: "Year",
      domain: yDomain
    },
    color: {
      ...colorScale,
      legend: true,
      label: "Unemployment Rate"
    },
    marks: [
      Plot.cell(heatmapData, {
        x: "County",
        y: "Year",
        fill: "Value",
        tip: d => `County: ${d.County}\nYear: ${d.Year}\nUnemployment Rate: ${d.Value}`
      })
    ]
  });
}

// Transform data into a long format suitable for heatmap
function transformData(rawData) {
  return rawData.flatMap(row => {
    const county = row.County;
    return Object.keys(row)
      .filter(key => key !== "County")
      .map(year => ({
        County: county,
        Year: parseInt(year),
        Value: parseFloat(row[year])
      }));
  });
}

// Load the CSV data
const rawData = await FileAttachment("UnemploymentRates.csv").csv({ typed: true });
const transformedData = transformData(rawData);

// Create a slider for year selection
const yearSlider = Inputs.range([
  d3.min(transformedData, d => d.Year),
  d3.max(transformedData, d => d.Year)
], {
  step: 1,
  label: "Select Year"
});

// Heatmap container
const heatmapContainer = document.createElement("div");

// Update the heatmap based on selected year
function updateHeatmap(selectedYear) {
  heatmapContainer.innerHTML = "";

  const filteredData = transformedData.filter(d => d.Year === selectedYear);

  if (filteredData.length === 0) {
    const noDataMessage = document.createElement("div");
    noDataMessage.textContent = `No data available for the year ${selectedYear}`;
    noDataMessage.style.color = "red";
    noDataMessage.style.textAlign = "center";
    heatmapContainer.appendChild(noDataMessage);
    return;
  }

  const heatmap = generateHeatmap(filteredData);
  heatmapContainer.appendChild(heatmap);
}

// Attach event listener to slider
yearSlider.addEventListener("input", () => {
  updateHeatmap(yearSlider.value);
});

// Initial render
updateHeatmap(yearSlider.value);

display(yearSlider);
display(heatmapContainer);

```