<script>
  import { onMount, createEventDispatcher } from 'svelte';
  import * as d3 from 'd3';
  
  // Dimensions.
  const width = 960, height = 600, padding = 20;
  let svgElement, svg;
  
  // Data holders.
  let geoData;            // Neighborhoods GeoJSON.
  let neighborhoods;      // Array of neighborhood features.
  let vacancyMap = {};    // Mapping: Neighborhood name -> Vacant Rate (decimal fraction).
  let neighborhoodDetails = {}; // Mapping: Neighborhood name -> { converted, others }
  let bipocMap = {};      // Mapping: Neighborhood name -> Bipoc Rate (decimal fraction).
  
  // Continuous color scale for vacancy.
  let colorScale;
  
  // Track the currently selected neighborhood.
  let selectedNeighborhood = null;
  
  // Default values for averaging (computed from CSV data).
  let defaultDetails = null;
  let defaultBipocRate = null;
  
  const dispatch = createEventDispatcher();
  
  // Create a default projection centered on Boston.
  function createDefaultProjection() {
    return d3.geoMercator()
      .scale(70000 * 2)
      .center([-71.0589, 42.3101])
      .translate([width / 2, height / 2]);
  }
  
  let projection = createDefaultProjection();
  let path = d3.geoPath().projection(projection);
  
  // Load GeoJSON and CSV.
  async function loadData() {
    try {
      // Load neighborhoods GeoJSON.
      geoData = await d3.json('data/neighborhoods.geojson');
      neighborhoods = geoData.features;
      
      // Load CSV of neighborhood details.
      const csvData = await d3.csv('data/Just Neighborhood Details.csv');
      csvData.forEach(d => {
        const name = d.Neighborhood.trim();
        vacancyMap[name] = +d["Vacant Rate"];
        // Conversion percentages: given as decimals, so multiply by 100.
        const twoFam = +d["% of Condoconversions that TWO-FAM DWELL"] * 100;
        const threeFam = +d["% of Condoconversions that THREE-FAM DWELL"] * 100;
        const converted = twoFam + threeFam;
        const others = 100 - converted;
        neighborhoodDetails[name] = { converted, others };
        bipocMap[name] = +d["Bipoc Rate"];
      });
      
      // Get unique neighborhood names.
      const neighborhoodNames = Array.from(new Set(
        neighborhoods.map(f => (f.properties.Neighborhood || f.properties.neighborhood || f.properties.name || "").trim())
          .filter(n => n !== "")
      ));
      
      // Compute the extent of vacancy rates.
      const vacancyRates = neighborhoodNames.map(n => vacancyMap[n]).filter(v => v != null);
      const vacancyExtent = d3.extent(vacancyRates);
      // Create a continuous color scale (darker red for higher vacancy).
      colorScale = d3.scaleSequential(d3.interpolateReds).domain(vacancyExtent);
      
      // Compute defaults (averages) for all neighborhoods.
      let totalConverted = 0, totalOthers = 0, totalBipoc = 0, count = 0;
      neighborhoodNames.forEach(n => {
        if (neighborhoodDetails[n]) {
          totalConverted += neighborhoodDetails[n].converted;
          totalOthers += neighborhoodDetails[n].others;
          count++;
        }
        if (bipocMap[n] != null) {
          totalBipoc += bipocMap[n];
        }
      });
      const avgConverted = count ? totalConverted / count : 0;
      const avgOthers = count ? totalOthers / count : 0;
      const avgBipoc = count ? totalBipoc / count : 0;
      defaultDetails = { converted: avgConverted, others: avgOthers };
      defaultBipocRate = avgBipoc;
      
      // Dispatch default values so the parent can display these before any selection.
      dispatch("selectNeighborhood", { name: "Average", details: defaultDetails, bipocRate: defaultBipocRate });
      
      drawMap();
    } catch (error) {
      console.error("Error loading data:", error);
    }
  }
  
  // Draw neighborhood polygons.
  function drawMap() {
    svg.selectAll('*').remove();
    svg.selectAll("path.neighborhood")
      .data(neighborhoods)
      .enter()
      .append("path")
      .attr("class", "neighborhood")
      .attr("d", path)
      .attr("fill", d => {
         const name = (d.properties.Neighborhood || d.properties.neighborhood || d.properties.name || "").trim();
         return vacancyMap[name] != null ? colorScale(vacancyMap[name]) : "#ccc";
      })
      .attr("stroke", "#333")
      .style("cursor", "pointer")
      .on("mouseover", function(event, d) {
         const currentFill = d3.select(this).attr("fill");
         d3.select(this).attr("fill", d3.color(currentFill).darker(0.2));
         // Optionally, add a tooltip.
         const name = (d.properties.Neighborhood || d.properties.neighborhood || d.properties.name || "").trim();
         d3.select(this).append("title").text(name);
      })
      .on("mouseout", function(event, d) {
         const name = (d.properties.Neighborhood || d.properties.neighborhood || d.properties.name || "").trim();
         d3.select(this).attr("fill", vacancyMap[name] != null ? colorScale(vacancyMap[name]) : "#ccc");
      })
      .on("click", (event, d) => {
         const name = (d.properties.Neighborhood || d.properties.neighborhood || d.properties.name || "").trim();
         if (!selectedNeighborhood || selectedNeighborhood !== d) {
           selectedNeighborhood = d;
           zoomIn(d);
           // Dispatch event with the selected neighborhood's details.
           dispatch("selectNeighborhood", {
             name,
             details: neighborhoodDetails[name],
             bipocRate: bipocMap[name]
           });
         } else {
           // When clicking the same neighborhood to reset, we zoom out and send default averages.
           selectedNeighborhood = null;
           resetZoom();
           dispatch("selectNeighborhood", {
             name: "Average",
             details: defaultDetails,
             bipocRate: defaultBipocRate
           });
         }
      });
  }
  
  function zoomIn(feature) {
    const t = d3.transition().duration(800);
    projection.fitExtent([[padding, padding], [width - padding, height - padding]], feature);
    path = d3.geoPath().projection(projection);
    svg.selectAll("path.neighborhood")
       .transition(t)
       .attr("d", path);
  }
  
  function resetZoom() {
    const t = d3.transition().duration(800);
    projection = createDefaultProjection();
    path = d3.geoPath().projection(projection);
    svg.selectAll("path.neighborhood")
       .transition(t)
       .attr("d", path);
  }
  
  onMount(async () => {
    svg = d3.select(svgElement)
            .attr("width", width)
            .attr("height", height);
    await loadData();
  });
</script>

<div style="display: flex; justify-content: center; align-items: flex-start;">
  <svg bind:this={svgElement} viewBox={`0 0 ${width} ${height}`}></svg>
  <!-- The parent component will listen for the "selectNeighborhood" event to update the PieChart and BipocIndicator -->
</div>

<style>
  svg {
    border: 1px solid #ccc;
  }
  path.neighborhood {
    cursor: pointer;
  }
</style>
