<script>
  import { onMount, afterUpdate } from 'svelte';
  import * as d3 from 'd3';
  
  export let details; // Expected shape: { converted, others }
  
  let container;
  const chartWidth = 300, chartHeight = 300;
  const radius = Math.min(chartWidth, chartHeight) / 2;
  
  // We'll define our legend data here so that it always matches our two slices.
  const legendData = [
    { label: "Homes", color: "#1f77b4" },
    { label: "Others", color: "#ff7f0e" }
  ];
  
  function drawPieChart() {
    if (!details) return;
    
    // Prepare pie data.
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
                 .attr("transform", `translate(${chartWidth/2}, ${chartHeight/2})`);
    } else {
      svgPie = svgPie.select("g");
      if (svgPie.empty()) {
        svgPie = d3.select(container).select("svg.pie-chart").append("g")
                    .attr("transform", `translate(${chartWidth/2}, ${chartHeight/2})`);
      }
    }
    
    const pie = d3.pie()
                  .sort(null)
                  .value(d => d.value);
    
    const arc = d3.arc()
                  .innerRadius(radius * 0.4)
                  .outerRadius(radius - 5)
                  .cornerRadius(5)
                  .padAngle(0.02);
    
    // Use fixed colors for the pie slices.
    const pieColor = d3.scaleOrdinal()
                       .domain(["Homes", "Others"])
                       .range(["#1f77b4", "#ff7f0e"]);
    
    // DATA JOIN for arcs.
    const arcs = svgPie.selectAll(".arc")
                       .data(pie(pieData), d => d.data.label);
    
    // EXIT
    arcs.exit().remove();
    
    // ENTER
    const arcsEnter = arcs.enter()
                          .append("g")
                          .attr("class", "arc");
    
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
             .style("font-size", "16px")       // increased size
             .style("font-weight", "bold")     // bold the text
             .style("fill", "#000"); // changed to black
    
    // MERGE + UPDATE
    const allArcs = arcsEnter.merge(arcs);
    
    allArcs.select("path")
           .transition()
           .duration(800)
           .attrTween("d", function(d) {
             const interpolate = d3.interpolate(this._current, d);
             this._current = interpolate(0);
             return function(t) {
               return arc(interpolate(t));
             };
           })
           .attr("fill", d => pieColor(d.data.label))
           .attr("fill-opacity", 0.7)
           .attr("stroke", d => pieColor(d.data.label))
           .attr("stroke-width", 5);
    
    allArcs.select("text")
           .transition()
           .duration(800)
           .attr("transform", d => `translate(${arc.centroid(d)})`)
           .text(d => `${Math.round(d.data.value)}%`)
           .style("font-size", "16px")       // increased size
           .style("font-weight", "bold")
           .style("fill", "#000"); // ensure text remains black
    
    drawLegend();
  }
  
  function drawLegend() {
    d3.select(container).select("svg.legend").remove();
    
    const legendWidth = 300, legendHeight = 50;
    
    const svgLegend = d3.select(container)
                        .append("svg")
                        .attr("class", "legend")
                        .attr("width", legendWidth)
                        .attr("height", legendHeight);
                        
    const legendItemHeight = 20;
    const legendItems = svgLegend.selectAll(".legend-item")
                                 .data(legendData)
                                 .enter()
                                 .append("g")
                                 .attr("class", "legend-item")
                                 .attr("transform", (d, i) => `translate(0, ${i * legendItemHeight})`);
    
    legendItems.append("rect")
               .attr("x", 10)
               .attr("y", 0)
               .attr("width", 15)
               .attr("height", 15)
               .attr("fill", d => d.color);
    
    legendItems.append("text")
               .attr("x", 30)
               .attr("y", 12)
               .text(d => d.label)
               .style("font-size", "12px")
               .style("fill", "#333");
  }
  
  onMount(() => {
    drawPieChart();
  });
  
  afterUpdate(() => {
    drawPieChart();
  });
</script>

<div>
  <h4 style="text-align: center; margin-bottom: 10px;">
    Percent of Condo Conversions That Were Homes
  </h4>
  <div bind:this={container}></div>
</div>

<style>
  div {
    text-align: center;
  }
  /*.pie-tooltip {
    position: absolute;
    text-align: center;
    padding: 5px;
    font: 12px sans-serif;
    background: rgba(0, 0, 0, 0.7);
    border-radius: 8px;
    pointer-events: none;
    color: #fff;
    opacity: 0;
  }*/
</style>
