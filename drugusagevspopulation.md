---
theme: dashboard
title: Population vs drug usage
toc: false
---
```js
const population=await FileAttachment("Population.csv").csv({typed: true});
const drugarrests=await FileAttachment("DrugArrests.csv").csv({typed: true});
```
```js
const populationLong = population.flatMap(row => {
  return Object.keys(row)
    .filter(key => key !== 'County')
    .map(year => ({
      County: row.County,
      Year: parseInt(year),
      Population: parseFloat(row[year])
    }));
});

const drugArrestsLong = drugarrests.flatMap(row => {
  return Object.keys(row)
    .filter(key => key !== 'County')
    .map(year => ({
      County: row.County,
      Year: parseInt(year),
      DrugArrests: parseFloat(row[year])
    }));
});

const mergedData = populationLong.map(pop => {
  const arrestData = drugArrestsLong.find(da => 
    da.County === pop.County && da.Year === pop.Year
  );
  return arrestData ? { ...pop, DrugArrests: arrestData.DrugArrests } : null;
}).filter(d => d);

display(mergedData);
```
```js
// Load the TopoJSON file
const us = await d3.json("https://cdn.jsdelivr.net/npm/us-atlas@3/counties-10m.json");

// Convert the TopoJSON counties object to GeoJSON
const counties = topojson.feature(us, us.objects.counties);

// Clean and normalize county names for matching
function cleanCountyName(name) {
  return name.trim().toLowerCase().replace(/ county$/, '');
}

// Filter counties that belong to West Virginia based on the FIPS prefix "54"
const westVirginiaCounties = counties.features.filter(
  county => county.id.startsWith("54")
);

// Create a dictionary to map county names to their IDs for West Virginia
const countyNameToId = westVirginiaCounties.reduce((acc, county) => {
  const name = cleanCountyName(county.properties.name);
  acc[name] = county.id;
  return acc;
}, {});

// Add the corresponding county ID to each row in the CSV data (West Virginia only)
population.forEach(row => {
  const countyName = cleanCountyName(row.County);
  row.id = countyNameToId[countyName] || null; // If no match found, set to null
});

// Create a dictionary to map county IDs to population data by year
// Create a dictionary to map county IDs to population data by year
const populationByYear = {};
const years = Object.keys(population[0]).filter(key => key.match(/^\d{4}$/));

years.forEach(year => {
  populationByYear[year] = population.reduce((acc, row) => {
    if (row.id && row[year] != null) {
      // Ensure row[year] is treated as a string before calling .replace()
      const value = row[year].toString().replace(/,/g, ''); // Convert to string, then remove commas
      acc[row.id] = +value; // Convert to a numeric value
    }
    return acc;
  }, {});
});

// Create HTML elements for slider and plot container


// Function to render the plot based on selected year
function renderChoroplethPlot(year) {
  const populationData = populationByYear[year]; // Ensure populationData is not null or undefined
  const plot = Plot.plot({
    width: 640,
    height: 400,
    projection: {
      type: "albers",
      domain: { type: "FeatureCollection", features: westVirginiaCounties }
    },
    marks: [
      Plot.geo(westVirginiaCounties, {
        fill: d => (populationData && d.id in populationData) ? populationData[d.id] : "#ddd", // Fill based on population value, default gray for missing
        stroke: "#000", // Add borders to counties for visibility
        strokeWidth: 0.5,
        tip: true
      })
    ],
    color: {
      type: "linear",
      scheme: "spectral",
      label: `${year} Population of County`,
      domain: populationData ? [0, d3.max(Object.values(populationData))] : [0, 1],
      unknown: "#ddd",
      legend: true
    }
  });
  const plotContainer = document.getElementById("plotContainer");
  plotContainer.innerHTML = "";
  resize(plot);
  plotContainer.appendChild(plot);
}
function renderPlots(year) {
  if (years.includes(year)) {
    renderChoroplethPlot(year);
  }
}

// Add event listener to update the plot when the year changes
const yearSlider = document.getElementById("year");
const yearLabel = document.getElementById("yearLabel");
yearSlider.min = years[0];
yearSlider.max = years[years.length - 1];
yearSlider.addEventListener("input", (event) => {
  const year = event.target.value;
  yearLabel.textContent = year;
  renderPlots(year);
});
```
```js
function renderDrugArrestsMultiLineChart(data) {
  return Plot.plot({
    title: "Drug Arrests Over Time by County",
    width: 400,
    height: 200,
    x: { label: "Year", grid: true },
    y: { label: "Drug Arrests", grid: true },
    color: { legend: true }, // Enable legend for counties
    marks: [
      Plot.line(data, {
        x: "Year",
        y: "DrugArrests",
        stroke: "County", // Assign a unique color per county
        strokeWidth: 1.0,
        title: d => `${d.County}, ${d.Year}: ${d.DrugArrests}`
      })
    ]
  });
}
d3.select("#line-chart-container").html(""); // Clear the container if needed
d3.select("#line-chart-container").append(() => renderDrugArrestsMultiLineChart(mergedData));
```
```js
function renderPopulationBarChart(data, selectedYear) {
  const filteredData = data.filter(d => d.Year === selectedYear);

  return Plot.plot({
    title: `Population by County (${selectedYear})`,
    width: 400,
    height: 200,
    x: {
      label: "County",
      domain: filteredData.map(d => d.County),
      tickRotate: -45,
      grid: true,
    },
    y: { label: "Population", grid: true },
    marks: [
      Plot.barY(filteredData, {
        x: "County",
        y: "Population",
        fill: "County",
        tip: true,
      }),
    ],
  });
}

// Create a year slider for the bar chart
const populationYears = Array.from(new Set(mergedData.map(d => d.Year))).sort();
const populationYearSlider = Inputs.range([Math.min(...populationYears), Math.max(...populationYears)], {
  label: "Select Year for Population Chart",
  step: 1,
  value: populationYears[0], // Default to the first year
});

// Append the slider and the bar chart container
d3.select("#barchart-container").html("");
display(populationYearSlider);

// Render the Population Bar Chart for the default year
function updatePopulationBarChart(selectedYear) {
  d3.select("#barchart-container").html("");
  const barChart = renderPopulationBarChart(mergedData, selectedYear);
  d3.select("#barchart-container").append(() => barChart);
}

updatePopulationBarChart(populationYearSlider.value);

// Add event listener for the slider to update the chart dynamically
populationYearSlider.addEventListener("input", () => {
  const selectedYear = parseInt(populationYearSlider.value);
  updatePopulationBarChart(selectedYear);
});
```
<div id="line-chart-container" style="border: 1px solid #ddd; padding: 10px;"></div>
<div id="barchart-container" style="border: 1px solid #ddd; padding: 10px;"></div>

 <div>
    <label for="year">Select a year:</label>
    <input type="range" id="year" step="1" value="${years[0]}" min="${years[0]}" max="${years[years.length - 1]}" oninput="yearLabel.textContent = this.value">
    <span id="yearLabel">${years[0]}</span>
  </div>
  <div id="plotsContainer" style="display: flex; justify-content: space-between; gap: 20px;">
  <div id="plotContainer">
  </div>
</div>
