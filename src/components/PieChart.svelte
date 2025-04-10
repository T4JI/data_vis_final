<script>
  import { onMount, afterUpdate } from 'svelte';
  import * as d3 from 'd3';
  
  export let details; // { converted, others }
  let container;
  const chartWidth = 450, chartHeight = 350; // Increased chart width
  const radius = Math.min(chartWidth, chartHeight) / 2;
  
  const legendData = [
    { label: "Homes", color: "#1f77b4" },
    { label: "Others", color: "#ff7f0e" }
  ];
  
  function drawPieChart() {
    if (1>2) {
      d3.select(container).selectAll("*").remove(); // Clear existing chart
      d3.select(container)
        .append("text")
        .attr("class", "no-data-text")
        .attr("x", chartWidth / 2)
        .attr("y", chartHeight / 2)
        .attr("text-anchor", "middle")
        .attr("dominant-baseline", "middle")
        .style("font-size", "18px")
        .style("font-weight", "bold")
        .style("fill", "#666")
        .text("No Data For This Region");
      return;
    }
    const pieData = [
      { label: "Converted Dwellings", value: details.converted },
      { label: "Others", value: details.others }
    ];
    let svgPie = d3.select(container).select("svg.pie-chart");
    if (svgPie.empty()) {
      svgPie = d3.select(container)
                 .append("svg")
                 .attr("class", "pie-chart")
                 .attr("width", chartWidth)
                 .attr("height", chartHeight)
                 .append("g")
                 .attr("transform", `translate(${chartWidth / 2}, ${chartHeight / 2})`);
    } else {
      svgPie = svgPie.select("g") || d3.select(container).select("svg.pie-chart").append("g")
                      .attr("transform", `translate(${chartWidth / 2}, ${chartHeight / 2})`);
    }
  
    const pie = d3.pie().sort(null).value(d => d.value);
    const arc = d3.arc()
                  .innerRadius(radius * 0.4)
                  .outerRadius(radius - 5)
                  .cornerRadius(5)
                  .padAngle(0.02);
  
    const pieColor = d3.scaleOrdinal()
                       .domain(["Homes", "Others"])
                       .range(["#1f77b4", "#ff7f0e"]);
  
    const arcs = svgPie.selectAll(".arc").data(pie(pieData), d => d.data.label);
    arcs.exit().remove();
  
    const arcsEnter = arcs.enter().append("g").attr("class", "arc");
    arcsEnter.append("path")
             .each(function(d) { this._current = d; })
             .attr("d", arc)
             .attr("fill", d => pieColor(d.data.label))
             .attr("fill-opacity", 0.7)
             .attr("stroke", "#fff")
             .attr("stroke-width", 1);
    arcsEnter.append("text")
             .attr("transform", d => `translate(${arc.centroid(d)})`)
             .attr("text-anchor", "middle")
             .attr("dy", "0.35em")
             .each(function(d) { this._current = d; })
             .text(d => `${Math.round(d.data.value)}%`)
             .style("font-size", "16px")
             .style("font-weight", "bold")
             .style("fill", "#000");
  
    const allArcs = arcsEnter.merge(arcs);
    allArcs.select("path")
           .transition().duration(800)
           .attrTween("d", function(d) {
             const interpolate = d3.interpolate(this._current, d);
             this._current = interpolate(0);
             return t => arc(interpolate(t));
           })
           .attr("fill", d => pieColor(d.data.label))
           .attr("fill-opacity", 0.7)
           .attr("stroke", d => pieColor(d.data.label))
           .attr("stroke-width", 5);
    allArcs.select("text")
           .transition().duration(800)
           .attr("transform", d => `translate(${arc.centroid(d)})`)
           .text(d => `${Math.round(d.data.value)}%`)
           .style("font-size", "16px")
           .style("font-weight", "bold")
           .style("fill", "#000");
  
    drawLegend();
  }
  
  function drawLegend() {
    d3.select(container).select("svg.legend").remove();
    const legendWidth = 200, legendHeight = 50 * legendData.length; // Adjust height for vertical stacking
    const svgLegend = d3.select(container)
                        .append("svg")
                        .attr("class", "legend")
                        .attr("width", legendWidth)
                        .attr("height", legendHeight)
                        .style("position", "relative")
                        .style("left", "-80px"); // Move legend 50px to the left
    const legendItems = svgLegend.selectAll(".legend-item")
                                 .data(legendData)
                                 .enter().append("g")
                                 .attr("class", "legend-item")
                                 .attr("transform", (d, i) => `translate(0, ${i * 20})`); // Stack items vertically
    legendItems.append("rect")
               .attr("x", 10)
               .attr("y", 10)
               .attr("width", 15)
               .attr("height", 15)
               .attr("fill", d => d.color);
    legendItems.append("text")
               .attr("x", 35)
               .attr("y", 22)
               .text(d => d.label)
               .style("font-size", "12px")
               .style("fill", "#333");
  }
  
  onMount(drawPieChart);
  afterUpdate(drawPieChart);
</script>
<div class="pie-chart-container">
  <h4>Percent of Conversions That Were Homes</h4>
  <div bind:this={container}></div>
</div>
<style>
  .pie-chart-container {
    text-align: center;
    display: flex;
    flex-direction: column;
    align-items: center; /* Center the pie chart horizontally */
    justify-content: center; /* Center the pie chart vertically */
    background-color: #f9f9f9;
    border-radius: 8px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    padding: 5px;
    margin-top: 5px;
    margin-bottom: -30px; /* Set a smaller margin underneath */
  }

  h4 {
    margin-bottom: 10px;
    font-weight: 400;
    color: #444;
  }
</style>
