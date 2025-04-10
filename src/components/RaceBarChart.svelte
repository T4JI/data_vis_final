<script>
  import { onMount, afterUpdate } from 'svelte';
  import * as d3 from 'd3';
  
  export let selectedNeighborhoodName = "";
  let csvData = [];
  let currentData = null;
  const raceCategories = ["Hisp Rate", "Asian Rate", "Black Rate", "White Rate", "Other and Two All Rate"];
  const colorScale = d3.scaleOrdinal()
                       .domain(raceCategories)
                       .range(d3.schemeSpectral[raceCategories.length])
                       .unknown("#ccc");
  const chartWidth = 400, chartHeight = 30;
  
  // Compute average for each category.
  function computeAverage(data) {
    const avg = {};
    raceCategories.forEach(cat => avg[cat] = d3.mean(data, d => +d[cat]));
    return avg;
  }
  
  function updateData() {
    const selectedRow = csvData.find(d => d.Neighborhood.trim() === selectedNeighborhoodName);
    currentData = selectedRow ? 
      Object.fromEntries(raceCategories.map(cat => [cat, +selectedRow[cat]])) :
      computeAverage(csvData);
  }
  
  function drawChart() {
    if (!csvData.length) return;
    updateData();
    let cumulative = 0;
    const segments = raceCategories.map(cat => {
      const value = currentData[cat];
      const start = cumulative;
      cumulative += value;
      return { category: cat, value, start, end: cumulative };
    });
    const dominantSegment = segments.reduce((max, seg) => seg.value > max.value ? seg : max, segments[0]);
    let svg = d3.select("#race-bar").select("svg.chart");
    if (svg.empty()) {
      svg = d3.select("#race-bar")
              .append("svg")
              .attr("class", "chart")
              .attr("width", chartWidth)
              .attr("height", chartHeight);
    }
    const seg = svg.selectAll("g.segment").data(segments, d => d.category);
    const segEnter = seg.enter().append("g").attr("class", "segment");
    segEnter.append("rect")
            .attr("x", d => d.start * chartWidth)
            .attr("y", 0)
            .attr("width", 0)
            .attr("height", chartHeight)
            .attr("fill", d => colorScale(d.category))
            .transition().duration(800)
            .attr("width", d => d.value * chartWidth)
            .attr("x", d => d.start * chartWidth);
    segEnter.append("text")
            .attr("x", d => (d.start + d.value / 2) * chartWidth)
            .attr("y", chartHeight / 2)
            .attr("text-anchor", "middle")
            .attr("dominant-baseline", "middle")
            .text(d => d === dominantSegment ? `${Math.round(d.value * 100)}%` : "")
            .style("opacity", d => d === dominantSegment ? 1 : 0)
            .style("fill", "#000")
            .style("font-size", d => d === dominantSegment ? "12px" : "10px") // Slightly larger for dominant
            .style("font-weight", d => d === dominantSegment ? "bold" : "normal"); // Bold for dominant
    const segMerge = segEnter.merge(seg);
    segMerge.select("rect")
            .transition().duration(800)
            .attr("x", d => d.start * chartWidth)
            .attr("width", d => d.value * chartWidth)
            .attr("fill", d => colorScale(d.category));
    segMerge.select("text")
            .transition().duration(800)
            .attr("x", d => (d.start + d.value / 2) * chartWidth)
            .text(d => d === dominantSegment ? `${Math.round(d.value * 100)}%` : "")
            .style("opacity", d => d === dominantSegment ? 1 : 0)
            .style("fill", "#000")
            .style("font-size", d => d === dominantSegment ? "12px" : "10px") // Slightly larger for dominant
            .style("font-weight", d => d === dominantSegment ? "bold" : "normal"); // Bold for dominant
    segMerge.on("mouseover", function(event, d) {
      d3.select(this).select("rect")
        .transition().duration(200)
        .attr("x", d => (d.start * chartWidth) - 2)
        .attr("width", d => (d.value * chartWidth) + 4)
        .attr("y", -6)
        .attr("height", chartHeight + 12);
      d3.select(this).select("text")
        .transition().duration(200)
        .style("opacity", 1)
        .style("fill", "#000") // Ensure text is black on hover
        .style("font-size", "16px")
        .style("font-weight", "bold")
        .text(`${Math.round(d.value * 100)}%`);
    })
    .on("mouseout", function(event, d) {
      d3.select(this).select("rect")
        .transition().duration(200)
        .attr("x", d => d.start * chartWidth)
        .attr("width", d => d.value * chartWidth)
        .attr("y", 0)
        .attr("height", chartHeight);
      d3.select(this).select("text")
        .transition().duration(200)
        .style("opacity", d === dominantSegment ? 1 : 0)
        .style("fill", "#000") // Ensure text is black after hover
        .style("font-size", "10px")
        .style("font-weight", "normal")
        .text(d === dominantSegment ? `${Math.round(d.value * 100)}%` : "");
    });
    seg.exit().remove();
    drawLegend();
  }
  
  function drawLegend() {
    d3.select("#race-bar").select("svg.legend").remove();
    const legendHeight = 40;
    const legendSvg = d3.select("#race-bar")
                         .append("svg")
                         .attr("class", "legend")
                         .attr("width", chartWidth)
                         .attr("height", legendHeight)
                         .style("display", "block")
                         .style("margin-top", "5px");
    const legendItemWidth = chartWidth / raceCategories.length;
    const legendLabels = ["Hisp. ", "Asian", "Black", "White", "Other"];
    const legendItems = legendSvg.selectAll("g.legend-item")
      .data(raceCategories)
      .enter().append("g")
      .attr("class", "legend-item")
      .attr("transform", (d,i)=> `translate(${i * legendItemWidth},0)`);
    legendItems.append("rect")
      .attr("x", 5)
      .attr("y", 5)
      .attr("width", 12)
      .attr("height", 12)
      .attr("fill", d => colorScale(d));
    legendItems.append("text")
      .attr("x", 22)
      .attr("y", 16)
      .text((d,i)=> legendLabels[i])
      .style("font-size", "14px")
      .style("fill", "#333")
      .style("font-weight", "bold");
  }
  
  onMount(async () => {
    csvData = await d3.csv('data/NeighborhoodsAndRaces.csv');
    drawChart();
  });
  afterUpdate(drawChart);
</script>

<div id="race-bar" class="race-bar-container"></div>

<style>
  .race-bar-container {
    text-align: center;
    margin: 20px auto; /* Center the race bar chart horizontally */
    max-width: 400px; /* Ensure it doesn't exceed the chart width */
  }
</style>
