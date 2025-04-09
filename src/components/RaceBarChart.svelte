<script>
  import { onMount, afterUpdate } from 'svelte';
  import * as d3 from 'd3';
  
  // Prop: selectedNeighborhoodName is a string;
  // if empty (or doesn't match any row), use the average across all neighborhoods.
  export let selectedNeighborhoodName = "";
  
  let csvData = [];       // Raw CSV data
  let currentData = null; // Object with race proportions for current selection.
  
  // Define the race categories (order matters)
  const raceCategories = ["Hisp Rate", "Black Rate", "Asian Rate", "White Rate", "Other and Two All Rate"];
  
  // Updated color scale using d3.schemeSpectral.
  const colorScale = d3.scaleOrdinal()
                       .domain(raceCategories)
                       .range(d3.schemeSpectral[raceCategories.length])
                       .unknown("#ccc");
  
  // Chart dimensions
  const chartWidth = 300, chartHeight = 30;
  
  // This function computes the average for each race category.
  function computeAverage(data) {
    let avg = {};
    raceCategories.forEach(cat => {
      avg[cat] = d3.mean(data, d => +d[cat]);
    });
    return avg;
  }
  
  // When new CSV data is loaded or selectedNeighborhoodName changes,
  // update the currentData.
  function updateData() {
    // Find the row that matches the selected neighborhood.
    const selectedRow = csvData.find(d => d.Neighborhood.trim() === selectedNeighborhoodName);
    if (selectedRow) {
      // Convert race values to numbers.
      currentData = {};
      raceCategories.forEach(cat => {
        currentData[cat] = +selectedRow[cat];
      });
    } else {
      // Use average values if no selection or no match.
      currentData = computeAverage(csvData);
    }
  }
  
  // Draw the stacked normalized bar chart.
  function drawChart() {
    if (!csvData.length) return; // Added guard: only run chart code if CSV data exists
    updateData();
    // Compute cumulative data for stacking.
    let cumulative = 0;
    // Create an array of segments.
    const segments = raceCategories.map(cat => {
      const value = currentData[cat];
      const start = cumulative;
      cumulative += value;
      return { category: cat, value, start, end: cumulative };
    });
    
    // Select or create an SVG.
    let svg = d3.select("#race-bar").select("svg.chart");
    if (svg.empty()) {
      svg = d3.select("#race-bar")
              .append("svg")
              .attr("class", "chart")
              .attr("width", chartWidth)
              .attr("height", chartHeight);
    }
    
    // Bind segments to group elements
    const seg = svg.selectAll("g.segment")
                   .data(segments, d => d.category);
    
    // ENTER: Create groups, append rect and text
    const segEnter = seg.enter()
                        .append("g")
                        .attr("class", "segment");
    
    segEnter.append("rect")
            .attr("x", d => d.start * chartWidth)
            .attr("y", 0)
            .attr("width", 0) // start with 0 width for animation
            .attr("height", chartHeight)
            .attr("fill", d => colorScale(d.category))
            .transition()
            .duration(800)
            .attr("width", d => d.value * chartWidth)
            .attr("x", d => d.start * chartWidth);
    
    segEnter.append("text")
            .attr("x", d => (d.start + d.value/2) * chartWidth)
            .attr("y", chartHeight/2)
            .attr("text-anchor", "middle")
            .attr("dominant-baseline", "middle")
            .text(d => `${(d.value * 100).toFixed(1)}%`)
            .style("opacity", 0)  // initially hidden
            .style("fill", "#fff")
            .style("font-size", "10px");
    
    // MERGE: Transition updated segments
    const segMerge = segEnter.merge(seg);
    
    segMerge.select("rect")
            .transition()
            .duration(800)
            .attr("x", d => d.start * chartWidth)
            .attr("width", d => d.value * chartWidth)
            .attr("fill", d => colorScale(d.category));
    
    segMerge.select("text")
            .transition()
            .duration(800)
            .attr("x", d => (d.start + d.value/2) * chartWidth)
            .text(d => `${(d.value * 100).toFixed(1)}%`);
    
    // Attach hover events to show percentage text (bold) and enlarge the bar.
    segMerge.on("mouseover", function(event, d) {
        d3.select(this).select("rect")
          .transition().duration(200)
          // Grow vertically and horizontally: expand width by 4px (2px on each side)
          .attr("x", d => (d.start * chartWidth) - 2)
          .attr("width", d => (d.value * chartWidth) + 4)
          .attr("y", -6)                     // increased vertical growth
          .attr("height", chartHeight + 12);  // increased vertical growth
        d3.select(this).select("text")
          .transition().duration(200)
          .style("opacity", 1)
          .style("fill", "#000")             // black text
          .style("font-size", "16px")        // larger text
          .style("font-weight", "bold")      // bold text
          .text(`${Math.round(d.value * 100)}%`); // rounded percentage
      })
      .on("mouseout", function(event, d) {
        d3.select(this).select("rect")
          .transition().duration(200)
          // Revert to original dimensions.
          .attr("x", d => d.start * chartWidth)
          .attr("width", d => d.value * chartWidth)
          .attr("y", 0)
          .attr("height", chartHeight);
        d3.select(this).select("text")
          .transition().duration(200)
          .style("opacity", 0)
          .style("font-size", "10px")
          .style("font-weight", "normal")
          .text(`${Math.round(d.value * 100)}%`);
      });
    
    seg.exit().remove();
    
    // Draw legend below the chart.
    drawLegend();
  }
  
  function drawLegend() {
    // Remove any existing legend.
    d3.select("#race-bar").select("svg.legend").remove();
    
    const legendHeight = 40;
    const legendSvg = d3.select("#race-bar")
                        .append("svg")
                        .attr("class", "legend")
                        .attr("width", chartWidth)
                        .attr("height", legendHeight)
                        .style("display", "block")
                        .style("margin-top", "5px");
    
    // Distribute legend items equally.
    const legendItemWidth = chartWidth / raceCategories.length;
    const legendLabels = ["Hisp. ", "Black", "Asian", "White", "Other"]; // Updated legend labels
    
    const legendItems = legendSvg.selectAll("g.legend-item")
      .data(raceCategories)
      .enter()
      .append("g")
      .attr("class", "legend-item")
      .attr("transform", (d, i) => `translate(${i * legendItemWidth}, 0)`);
      
    legendItems.append("rect")
      .attr("x", 5)
      .attr("y", 5)
      .attr("width", 12)
      .attr("height", 12)
      .attr("fill", d => colorScale(d));
      
    legendItems.append("text")
      .attr("x", 22)
      .attr("y", 16)
      .text((d, i) => legendLabels[i])
      .style("font-size", "14px") // bigger text
      .style("fill", "#333")
      .style("font-weight", "bold"); // bold text
  }
  
  onMount(async () => {
    // Load CSV data.
    csvData = await d3.csv('data/NeighborhoodsAndRaces.csv');
    drawChart();
  });
  
  afterUpdate(() => {
    drawChart();
  });
</script>

<div id="race-bar" style="width:100%; max-width:300px; height:30px; margin: 10px auto;"></div>

<style>
  #race-bar {
    text-align: center;
  }
</style>
