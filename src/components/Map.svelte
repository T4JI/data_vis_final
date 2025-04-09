<script>
  import { onMount, createEventDispatcher } from 'svelte';
  import * as d3 from 'd3';
  import * as turf from '@turf/turf';  // for random point generation
  
  // Dimensions and SVG container.
  const width = 960, height = 600, padding = 20;
  let svgElement, svg;
  
  // Data holders.
  let geoData, neighborhoods;
  let vacancyMap = {}, neighborhoodDetails = {}, bipocMap = {};
  let colorScale, vacancyExtentGlobal;
  let selectedNeighborhood = null;
  let defaultDetails = null, defaultBipocRate = null;
  
  const dispatch = createEventDispatcher();
  
  // Projection and path generator.
  function createDefaultProjection() {
    return d3.geoMercator()
             .scale(140000)  // simplified factor (70000*2)
             .center([-71.0589, 42.3101])
             .translate([width / 2, height / 2]);
  }
  let projection = createDefaultProjection();
  let path = d3.geoPath().projection(projection);
  
  // Helper: Generate points within a polygon using Turf.
  function generatePoints(feature, count) {
    const points = [];
    const bbox = turf.bbox(feature);
    let attempts = 0;
    while (points.length < count && attempts < 100) {
      const pt = turf.randomPoint(1, { bbox }).features[0];
      if (turf.booleanPointInPolygon(pt, feature)) points.push(pt.geometry.coordinates);
      attempts++;
    }
    return points;
  }
  
  // Load GeoJSON and CSV, compute defaults and color scale.
  async function loadData() {
    try {
      geoData = await d3.json('data/neighborhoods.geojson');
      neighborhoods = geoData.features;
      const csvData = await d3.csv('data/Just Neighborhood Details.csv');
      csvData.forEach(d => {
        const name = d.Neighborhood.trim();
        vacancyMap[name] = +d["Vacant Rate"];
        const twoFam = +d["% of Condoconversions that TWO-FAM DWELL"] * 100;
        const threeFam = +d["% of Condoconversions that THREE-FAM DWELL"] * 100;
        neighborhoodDetails[name] = { converted: twoFam + threeFam, others: 100 - (twoFam + threeFam) };
        bipocMap[name] = +d["Bipoc Rate"];
      });
      const neighborhoodNames = Array.from(new Set(
        neighborhoods.map(f => (f.properties.Neighborhood || f.properties.neighborhood || f.properties.name || "").trim()).filter(n => n)
      ));
      const vacancyRates = neighborhoodNames.map(n => vacancyMap[n]).filter(v => v != null);
      vacancyExtentGlobal = d3.extent(vacancyRates);
      colorScale = d3.scaleSequential(d3.interpolateReds).domain(vacancyExtentGlobal);
  
      // Compute defaults.
      let totalConverted = 0, totalOthers = 0, totalBipoc = 0, count = 0;
      neighborhoodNames.forEach(n => {
        if (neighborhoodDetails[n]) {
          totalConverted += neighborhoodDetails[n].converted;
          totalOthers += neighborhoodDetails[n].others;
          count++;
        }
        if (bipocMap[n] != null) totalBipoc += bipocMap[n];
      });
      const avgConverted = count ? totalConverted / count : 0;
      const avgOthers = count ? totalOthers / count : 0;
      defaultDetails = { converted: avgConverted, others: avgOthers };
      defaultBipocRate = count ? totalBipoc / count : 0;
      dispatch("selectNeighborhood", { name: "Average", details: defaultDetails, bipocRate: defaultBipocRate });
      drawMap();
    } catch (e) {
      console.error("Error loading data:", e);
    }
  }
  
  // Common mouse event handlers.
  function handleMouseOver(event, d, el) {
    d3.select(el).attr("fill", d3.color(d3.select(el).attr("fill")).darker(0.2));
    const name = (d.properties.Neighborhood || d.properties.neighborhood || d.properties.name || "").trim();
    const centroid = path.centroid(d);
    svg.append("text")
       .attr("class", "neighborhood-label")
       .attr("x", centroid[0])
       .attr("y", centroid[1])
       .attr("text-anchor", "middle")
       .attr("dominant-baseline", "middle")
       .style("fill", "#000")
       .style("font-size", "14px")
       .style("font-weight", "bold")
       .style("pointer-events", "none")
       .style("opacity", 0)
       .text(name)
       .transition().duration(200).style("opacity", 1);
  }
  function handleMouseOut(event, d, el) {
    svg.selectAll("text.neighborhood-label").remove();
    const name = (d.properties.Neighborhood || d.properties.neighborhood || d.properties.name || "").trim();
    d3.select(el).attr("fill", vacancyMap[name] != null ? colorScale(vacancyMap[name]) : "#ccc");
  }
  
  // Zoom functions.
  function zoomIn(feature) {
    const t = d3.transition().duration(800);
    projection.fitExtent([[padding, padding], [width - padding, height - padding]], feature);
    path = d3.geoPath().projection(projection);
    svg.selectAll("path.neighborhood").transition(t).attr("d", path);
  }
  function resetZoom() {
    const t = d3.transition().duration(800);
    projection = createDefaultProjection();
    path = d3.geoPath().projection(projection);
    svg.selectAll("path.neighborhood").transition(t).attr("d", path);
  }
  
  // Draw the map: neighborhoods and icons.
  function drawMap() {
    svg.selectAll("*").remove();
    svg.selectAll("path.neighborhood")
       .data(neighborhoods)
       .enter()
       .append("path")
       .attr("class", "neighborhood")
       .attr("d", path)
       .attr("fill", d => {
         const n = (d.properties.Neighborhood || d.properties.neighborhood || d.properties.name || "").trim();
         return vacancyMap[n] != null ? colorScale(vacancyMap[n]) : "#ccc";
       })
       .attr("stroke", "#333")
       .style("cursor", "pointer")
       .on("mouseover", (event, d) => handleMouseOver(event, d, event.currentTarget))
       .on("mouseout", (event, d) => handleMouseOut(event, d, event.currentTarget))
       .on("click", (event, d) => {
           svg.selectAll("text.neighborhood-label").remove();
           const n = (d.properties.Neighborhood || d.properties.neighborhood || d.properties.name || "").trim();
           if (!selectedNeighborhood || selectedNeighborhood !== d) {
             selectedNeighborhood = d;
             zoomIn(d);
             dispatch("selectNeighborhood", { name: n, details: neighborhoodDetails[n], bipocRate: bipocMap[n] });
           } else {
             selectedNeighborhood = null;
             resetZoom();
             dispatch("selectNeighborhood", { name: "Average", details: defaultDetails, bipocRate: defaultBipocRate });
           }
       });
    // Append icons based on vacancy percentage.
    svg.selectAll("g.icon-group")
       .data(neighborhoods)
       .enter()
       .append("g")
       .attr("class", "icon-group")
       .each(function(d) {
         const n = (d.properties.Neighborhood || d.properties.neighborhood || d.properties.name || "").trim();
         const rate = vacancyMap[n] != null ? vacancyMap[n] : 0;
         const iconCount = Math.max(Math.ceil(rate * 10), 1);
         let pts = generatePoints(d, iconCount);
         if (pts.length === 0) pts = [path.centroid(d)];
         d3.select(this).selectAll("circle.icon")
           .data(pts)
           .enter()
           .append("circle")
           .attr("class", "icon")
           .attr("cx", p => p[0])
           .attr("cy", p => p[1])
           .attr("r", 4)
           .style("fill", "blue")
           .style("stroke", "#fff")
           .style("stroke-width", "1px")
           .on("mouseover", function() {
             d3.select(this).transition().duration(100).attr("r", 8);
           })
           .on("mouseout", function() {
             d3.select(this).transition().duration(100).attr("r", 4);
           });
       });
    drawMapLegend();
  }
  
  // Draw a continuous color legend.
  function drawMapLegend() {
    d3.select("#map-legend").remove();
    const legendWidth = 200, legendHeight = 10;
    let defs = svg.select("defs");
    if (defs.empty()) { defs = svg.append("defs"); }
    const gradient = defs.append("linearGradient").attr("id", "legend-gradient");
    const ticks = d3.ticks(vacancyExtentGlobal[0], vacancyExtentGlobal[1], 10);
    gradient.selectAll("stop")
      .data(ticks)
      .enter()
      .append("stop")
      .attr("offset", d => (((d - vacancyExtentGlobal[0]) / (vacancyExtentGlobal[1] - vacancyExtentGlobal[0])) * 100) + "%")
      .attr("stop-color", d => colorScale(d));
  
    const legendGroup = svg.append("g")
      .attr("id", "map-legend")
      .attr("transform", `translate(${width - legendWidth - 30}, ${height - 40})`);
    legendGroup.append("rect")
      .attr("width", legendWidth)
      .attr("height", legendHeight)
      .style("fill", "url(#legend-gradient)");
  
    const xScale = d3.scaleLinear().domain(vacancyExtentGlobal).range([0, legendWidth]);
    const xAxis = d3.axisBottom(xScale).ticks(5).tickFormat(d3.format(".0%"));
    legendGroup.append("g")
      .attr("transform", `translate(0, ${legendHeight})`)
      .call(xAxis)
      .select(".domain").remove();
    legendGroup.append("text")
      .attr("x", legendWidth / 2)
      .attr("y", -5)
      .attr("text-anchor", "middle")
      .style("font-size", "12px")
      .style("fill", "#333")
      .text("Vacancy Percentage");
  }
  
  onMount(async () => {
    svg = d3.select(svgElement).attr("width", width).attr("height", height);
    await loadData();
  });
</script>

<div style="display: flex; justify-content: center; align-items: flex-start;">
  <svg bind:this={svgElement} viewBox={`0 0 ${width} ${height}`}></svg>
  <!-- The parent component will listen for the "selectNeighborhood" event to update the PieChart and BipocIndicator -->
</div>

<style>
  svg { border: 1px solid #ccc; }
  /* ...existing styles... */
</style>
