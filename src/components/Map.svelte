<script>
  import { onMount, createEventDispatcher } from 'svelte';
  import * as d3 from 'd3';
  import * as turf from '@turf/turf';  // for random point generation
  
  // Dimensions and SVG container.
  const width = 960, height = 600, padding = 20;
  let svgElement, svg;
  
  // Data holders.
  let geoData, neighborhoods;
  let vacancyMap = {}, neighborhoodDetails = {}, bipocMap = {}, simpleConversion = {}; // new: add simpleConversion mapping
  let colorScale, vacancyExtentGlobal;
  let selectedNeighborhood = null;
  let defaultDetails = null, defaultBipocRate = null;
  let detailedView = false; // new: track if in detailed view
  
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
      // NEW: load simple conversion data
      const zipcodeData = await d3.csv('data/Neighborhood Zipcode Most Details.csv');
      zipcodeData.forEach(d => {
        const name = d.Neighborhood.trim();
        // The CSV value is like "0.18" meaning 18%, so multiply by 100.
        simpleConversion[name] = parseFloat(d["Simple Condo Conversion %"]) * 100;
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
  
  // Add a function to fade clouds in or out.
  function fadeClouds(opacity) {
    svg.selectAll("g.clouds")
      .transition()
      .duration(800)
      .style("opacity", opacity);
  }

  // Modify fadeLegend to handle both fading in and out with a smooth transition.
  function fadeLegend(opacity) {
    const legend = svg.select("#map-legend");
    if (!legend.empty()) {
      legend.transition()
        .duration(800)
        .style("opacity", opacity);
    }
  }
  
  // Zoom functions.
  function zoomIn(feature) {
    const t = d3.transition().duration(800);
    projection.fitExtent([[padding, padding], [width - padding, height - padding]], feature);
    path = d3.geoPath().projection(projection);
    svg.selectAll("path.neighborhood").transition(t).attr("d", path);
  
    fadeClouds(0); // Fade clouds out when zooming in.
    fadeLegend(0); // Fade legend out when zooming in.
  }
  function resetZoom() {
    const t = d3.transition().duration(800);
    projection = createDefaultProjection();
    path = d3.geoPath().projection(projection);
    svg.selectAll("path.neighborhood").transition(t).attr("d", path);
  
    fadeClouds(1); // Fade clouds in when resetting zoom.
    fadeLegend(1); // Fade legend back in smoothly when resetting zoom.

    // Redraw the legend after resetting the zoom.
    drawMapLegend();
  }
  
  // Draw the map: neighborhoods and icons.
  function drawMap() {
    svg.selectAll("*").remove();
    // Create a group for the map view.
    const mapGroup = svg.append("g").attr("class", "map-view").style("opacity", 1);
    
    mapGroup.selectAll("path.neighborhood")
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
         const n = (d.properties.Neighborhood || d.properties.neighborhood || d.properties.name || "").trim();
         if (!detailedView) {
           detailedView = true;
           // Dispatch selected neighborhood to update other vis.
           dispatch("selectNeighborhood", { name: n, details: neighborhoodDetails[n], bipocRate: bipocMap[n] });
           // Zoom in on selected neighborhood.
           zoomIn(d);
           // Fade out the entire map view.
           mapGroup.transition().duration(600).style("opacity", 0)
             .on("end", () => {
               mapGroup.remove();
               showDetailPlaceholder(d);
             });
         }
      });
    // Append icons based on vacancy percentage.
    mapGroup.selectAll("g.icon-group")
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
    // Redraw the legend.
    drawMapLegend();
  }
  
  // Modify drawMapLegend to ensure it starts with opacity 0 and fades in.
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
      .attr("transform", `translate(${width - legendWidth - 30}, ${height - 40})`)
      .style("opacity", 0); // Start with opacity 0.
  
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
  
    // Fade the legend in smoothly.
    legendGroup.transition()
      .duration(800)
      .style("opacity", 1);
  }
  
  // Add a new helper function to draw the detail placeholder.
  function showDetailPlaceholder(d) {
    svg.selectAll("*").remove();
    const n = (d.properties.Neighborhood || d.properties.neighborhood || d.properties.name || "").trim();
    const conversionPercent = simpleConversion[n] != null ? simpleConversion[n] : (neighborhoodDetails[n]?.converted || 0);
  
    const bounds = path.bounds(d); // Get the bounding box of the neighborhood.
    const x0 = bounds[0][0], y0 = bounds[0][1], x1 = bounds[1][0], y1 = bounds[1][1];
  
    const detailGroup = svg.append("g")
      .attr("class", "detail-placeholder")
      .attr("transform", "translate(0,30) scale(0.9)");
  
    detailGroup.append("path")
      .attr("d", path(d))
      .attr("fill", "none") // No fill for the neighborhood shape.
      .attr("stroke", "#333");
  
    detailGroup.append("text")
      .attr("x", (x0 + x1) / 2)
      .attr("y", y0 - 10)
      .attr("text-anchor", "middle")
      .style("font-size", "24px")
      .style("fill", "#000")
      .text(`${n}: ${conversionPercent}% of Buildings Now Condos`);
  
    // Generate multiple icons randomly distributed across the neighborhood.
    const iconCount = 100; // Number of icons to generate.
    const xMin = bounds[0][0], yMin = bounds[0][1];
    const xMax = bounds[1][0], yMax = bounds[1][1];
  
    const isPointInNeighborhood = (x, y) => {
      const geoPoint = projection.invert([x, y]); // Convert screen coordinates to geographic coordinates.
      return turf.booleanPointInPolygon({ type: "Point", coordinates: geoPoint }, d);
    };
  
    const homePercent = conversionPercent; // Percentage of conversions that were homes.
    const nonHomePercent = 100 - homePercent; // Percentage of conversions that were not homes.
    const homeCount = Math.round(iconCount * (homePercent / 100));
    const nonHomeCount = iconCount - homeCount;
  
    const shadowPercent = conversionPercent; // Percentage of icons with drop shadows.
    const shadowCount = Math.round(iconCount * (shadowPercent / 100));
    const shadowHomeCount = Math.round(shadowCount * (homePercent / 100)); // Drop shadows for homes.
    const shadowNonHomeCount = shadowCount - shadowHomeCount; // Drop shadows for non-homes.
  
    let shadowHomeCounter = 0;
    let shadowNonHomeCounter = 0;
  
    for (let i = 0; i < iconCount; i++) {
      let iconX, iconY;
  
      // Generate random positions within the bounding box until the point is inside the neighborhood.
      do {
        iconX = Math.random() * (xMax - xMin) + xMin;
        iconY = Math.random() * (yMax - yMin) + yMin;
      } while (!isPointInNeighborhood(iconX, iconY));
  
      // Add a low-opacity circle as a drop shadow for only `shadowCount` icons.
      if (shadowHomeCounter < shadowHomeCount && i < homeCount) {
        detailGroup.append("circle")
          .attr("cx", iconX)
          .attr("cy", iconY)
          .attr("r", 20)
          .attr("fill", "#ff7f0e") // Blue for homes.
          .attr("opacity", 0.3);
        shadowHomeCounter++;
      } else if (shadowNonHomeCounter < shadowNonHomeCount && i >= homeCount) {
        detailGroup.append("circle")
          .attr("cx", iconX)
          .attr("cy", iconY)
          .attr("r", 20)
          .attr("fill", "#1f77b4") // Orange for non-homes.
          .attr("opacity", 0.3);
        shadowNonHomeCounter++;
      }
  
      // Add the home icon.
      detailGroup.append("image")
        .attr("class", "center-icon")
        .attr("href", "data/home_icon2.gif")
        .attr("width", 70)
        .attr("height", 70)
        .attr("x", iconX - 35) // Center the icon at the point.
        .attr("y", iconY - 35)
        .on("mouseover", function () {
          d3.select(this).transition().duration(200).attr("width", 90).attr("height", 90).attr("x", iconX - 45).attr("y", iconY - 45); // Increase size on hover.
        })
        .on("mouseout", function () {
          d3.select(this).transition().duration(200).attr("width", 70).attr("height", 70).attr("x", iconX - 35).attr("y", iconY - 35); // Restore size.
        });
    }
  
    detailGroup.transition().duration(600).style("opacity", 1);
  
    // When clicking, transition with a zoom-out that removes the offset.
    detailGroup.style("cursor", "pointer")
      .on("click", () => {
        const centerX = (x0 + x1) / 2;
        const centerY = (y0 + y1) / 2 + 30; // Include vertical offset.
        detailGroup.transition().duration(600)
          .attr("transform", `translate(${centerX}, ${centerY}) scale(0)`)
          .on("end", () => {
            resetZoom();
            detailedView = false;
            drawMapWithFadeIn();
            dispatch("selectNeighborhood", { name: "Average", details: defaultDetails, bipocRate: defaultBipocRate });
          });
      });
  }
  
  // Add new helper function:
  function drawMapWithFadeIn() {
    svg.selectAll("*").remove();
    const mapGroup = svg.append("g").attr("class", "map-view").style("opacity", 0);
    
    mapGroup.selectAll("path.neighborhood")
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
        const n = (d.properties.Neighborhood || d.properties.neighborhood || d.properties.name || "").trim();
        if (!detailedView) {
          detailedView = true;
          removeClouds(); // Remove clouds when zooming in.
          dispatch("selectNeighborhood", { name: n, details: neighborhoodDetails[n], bipocRate: bipocMap[n] });
          zoomIn(d);
          mapGroup.transition().duration(600).style("opacity", 0)
            .on("end", () => {
              mapGroup.remove();
              showDetailPlaceholder(d);
            });
        }
      });
    
    // Add clouds to the zoomed-out view.
    drawClouds();
    
    // Redraw the legend when the map is reset.
    drawMapLegend();

    mapGroup.transition().duration(600).style("opacity", 1);
  }
  
  // Modify the drawClouds function to improve cloud appearance.
  function drawClouds() {
    const cloudGroup = svg.append("g").attr("class", "clouds").style("opacity", 1);
  
    const cloudData = [
      { x: -200, y: 50, width: 150, height: 80 },
      { x: 300, y: 100, width: 200, height: 100 },
      { x: 600, y: 200, width: 180, height: 90 },
      { x: -400, y: 150, width: 220, height: 110 },
    ];
  
    cloudGroup.selectAll("g.cloud")
      .data(cloudData)
      .enter()
      .append("g")
      .attr("class", "cloud")
      .each(function (d) {
        const cloud = d3.select(this);
  
        // Add multiple ellipses to create a fluffy cloud effect.
        cloud.append("ellipse")
          .attr("cx", d.x)
          .attr("cy", d.y)
          .attr("rx", d.width / 2)
          .attr("ry", d.height / 2)
          .attr("fill", "#D3D3D3") // Light blue color for clouds.
          .attr("stroke", "#D3D3D3") // Add stroke to clouds.
          .attr("stroke-width", 2)
          .attr("opacity", 0.9)
          .on("mouseover", function () {
            d3.select(this).transition().duration(200).attr("opacity", 0.5); // Reduce opacity on hover.
          })
          .on("mouseout", function () {
            d3.select(this).transition().duration(200).attr("opacity", 0.9); // Restore opacity.
          });
  
        cloud.append("ellipse")
          .attr("cx", d.x + d.width * 0.3)
          .attr("cy", d.y - d.height * 0.2)
          .attr("rx", d.width * 0.6 / 2)
          .attr("ry", d.height * 0.6 / 2)
          .attr("fill", "#D3D3D3")
          .attr("stroke", "#D3D3D3")
          .attr("stroke-width", 2)
          .attr("opacity", 0.8)
          .on("mouseover", function () {
            d3.select(this).transition().duration(200).attr("opacity", 0.5);
          })
          .on("mouseout", function () {
            d3.select(this).transition().duration(200).attr("opacity", 0.8);
          });
  
        cloud.append("ellipse")
          .attr("cx", d.x - d.width * 0.3)
          .attr("cy", d.y - d.height * 0.2)
          .attr("rx", d.width * 0.5 / 2)
          .attr("ry", d.height * 0.5 / 2)
          .attr("fill", "#D3D3D3")
          .attr("stroke", "#D3D3D3")
          .attr("stroke-width", 2)
          .attr("opacity", 0.7)
          .on("mouseover", function () {
            d3.select(this).transition().duration(200).attr("opacity", 0.5);
          })
          .on("mouseout", function () {
            d3.select(this).transition().duration(200).attr("opacity", 0.7);
          });
      });
  
    // Animate the clouds to move horizontally.
    function animateClouds() {
      cloudGroup.selectAll("g.cloud")
        .transition()
        .duration(30000)
        .ease(d3.easeLinear)
        .attr("transform", d => `translate(${d.x + 800}, ${d.y})`)
        .on("end", function (d) {
          d3.select(this).attr("transform", `translate(${d.x - 800}, ${d.y})`); // Reset position after moving off-screen.
          animateClouds(); // Restart animation.
        });
    }
  
    animateClouds();
  }
  
  // Add a function to remove clouds.
  function removeClouds() {
    svg.selectAll("g.clouds").remove();
  }
  
  // Ensure clouds are drawn by default when the page loads.
  onMount(async () => {
    svg = d3.select(svgElement).attr("width", width).attr("height", height);
  
    const defs = svg.append("defs");
    const filter = defs.append("filter").attr("id", "cloud-shadow").attr("x", "-50%").attr("y", "-50%").attr("width", "200%").attr("height", "200%");
    filter.append("feDropShadow")
      .attr("dx", 5)
      .attr("dy", 5)
      .attr("stdDeviation", 10)
      .attr("flood-color", "#000")
      .attr("flood-opacity", 0.2);
  
    await loadData();
  
    // Draw clouds in the zoomed-out view by default.
    drawClouds();
  });
</script>

<div style="display: flex; justify-content: center; align-items: flex-start;">
  <svg bind:this={svgElement} viewBox={`0 0 ${width} ${height}`}></svg>
  <!-- The parent component will listen for the "selectNeighborhood" event to update the PieChart and BipocIndicator -->
</div>

<style>
  svg {
    /* Remove the border */
    border: none;
  }
  /* ...existing styles... */
</style>
