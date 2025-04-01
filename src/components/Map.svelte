<script>
    import { onMount } from 'svelte';
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
  
    // We'll use a continuous color scale for vacancy.
    let colorScale;
    
    // Track the currently selected neighborhood.
    let selectedNeighborhood = null;
    
    // Create a default projection centered on Boston.
    function createDefaultProjection() {
      return d3.geoMercator()
        .scale(70000*2)
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
        .on("click", (event, d) => {
           const name = (d.properties.Neighborhood || d.properties.neighborhood || d.properties.name || "").trim();
           if (!selectedNeighborhood || selectedNeighborhood !== d) {
             selectedNeighborhood = d;
             zoomIn(d);
             updatePieChart(name);
             updateBipocIndicator(name);
           } else {
             selectedNeighborhood = null;
             resetZoom();
             removePieChart();
             removeBipocIndicator();
           }
        });
    }
    
    // Zoom in using projection.fitExtent.
    function zoomIn(feature) {
      const t = d3.transition().duration(800);
      projection.fitExtent([[padding, padding], [width - padding, height - padding]], feature);
      path = d3.geoPath().projection(projection);
      svg.selectAll("path.neighborhood")
         .transition(t)
         .attr("d", path);
    }
    
    // Reset zoom.
    function resetZoom() {
      const t = d3.transition().duration(800);
      projection = createDefaultProjection();
      path = d3.geoPath().projection(projection);
      svg.selectAll("path.neighborhood")
         .transition(t)
         .attr("d", path);
    }
    
    // Draw or update the pie chart.
    // The pie chart shows two slices: "Converted Dwellings" (combined 2- and 3-fam)
    // and "Others" (100 minus converted).
    function updatePieChart(neighborhoodName) {
      const details = neighborhoodDetails[neighborhoodName];
      if (!details) return;
      
      const pieData = [
        { label: "Converted Dwellings", value: details.converted },
        { label: "Others", value: details.others }
      ];
      
      const chartWidth = 300, chartHeight = 300;
      const radius = Math.min(chartWidth, chartHeight) / 2;
      
      // Select or create the pie chart SVG.
      let chartSvg = d3.select("#chart").select("svg");
      if (chartSvg.empty()) {
        chartSvg = d3.select("#chart")
                     .append("svg")
                     .attr("width", chartWidth)
                     .attr("height", chartHeight)
                     .append("g")
                     .attr("transform", `translate(${chartWidth/2}, ${chartHeight/2})`);
      } else {
        chartSvg = chartSvg.select("g");
        if (chartSvg.empty()) {
          chartSvg = d3.select("#chart").select("svg").append("g")
                       .attr("transform", `translate(${chartWidth/2}, ${chartHeight/2})`);
        }
      }
      
      const pie = d3.pie()
                    .sort(null)
                    .value(d => d.value);
      
      const arc = d3.arc()
                    .innerRadius(0)
                    .outerRadius(radius);
      
      // Use a fixed color scale for the pie slices.
      const pieColor = d3.scaleOrdinal()
                         .domain(["Converted Dwellings", "Others"])
                         .range(["#1f77b4", "#ff7f0e"]);
      
      // Bind data.
      const arcs = chartSvg.selectAll(".arc")
                           .data(pie(pieData));
      
      // UPDATE existing arcs.
      arcs.select("path")
          .transition().duration(800)
          .attr("d", arc)
          .attr("fill", d => pieColor(d.data.label));
      
      arcs.select("text")
          .transition().duration(800)
          .attr("transform", d => `translate(${arc.centroid(d)})`)
          .text(d => `${d.data.label}: ${d.data.value.toFixed(1)}%`);
      
      // ENTER new arcs.
      const arcsEnter = arcs.enter()
                            .append("g")
                            .attr("class", "arc");
      
      arcsEnter.append("path")
               .attr("d", arc)
               .attr("fill", d => pieColor(d.data.label));
      
      arcsEnter.append("text")
               .attr("transform", d => `translate(${arc.centroid(d)})`)
               .attr("text-anchor", "middle")
               .attr("dy", "0.35em")
               .text(d => `${d.data.label}: ${d.data.value.toFixed(1)}%`);
      
      // EXIT removed arcs.
      arcs.exit().remove();
    }
    
    // Remove pie chart.
    function removePieChart() {
      d3.select("#chart").selectAll("*").remove();
    }
    
    // Draw or update the BIPOC indicator.
    // We simulate a black silhouette (here a black rectangle) that gets filled from left to right
    // based on the bipoc rate (0 to 1). The fill overlay uses a clip path that transitions smoothly.
    function updateBipocIndicator(neighborhoodName) {
      removeBipocIndicator();
      const bipocRate = bipocMap[neighborhoodName];
      if (bipocRate == null) return;
      
      const indicatorWidth = 300, indicatorHeight = 50;
      
      // Append an SVG in the #bipoc container.
      const bipocSvg = d3.select("#bipoc")
                         .append("svg")
                         .attr("width", indicatorWidth)
                         .attr("height", indicatorHeight);
      
      // Append a black rectangle to simulate the silhouette.
      bipocSvg.append("rect")
              .attr("class", "bipoc-silhouette")
              .attr("x", 0)
              .attr("y", 0)
              .attr("width", indicatorWidth)
              .attr("height", indicatorHeight)
              .attr("fill", "#000");
      
      // Define a clip path that will reveal a colored overlay.
      const clip = bipocSvg.append("clipPath")
                           .attr("id", "bipocClip")
                           .append("rect")
                           .attr("x", 0)
                           .attr("y", 0)
                           .attr("width", 0)  // start with 0.
                           .attr("height", indicatorHeight);
      
      // Append a colored rectangle that uses the clip path.
      bipocSvg.append("rect")
              .attr("class", "bipoc-fill")
              .attr("x", 0)
              .attr("y", 0)
              .attr("width", indicatorWidth)
              .attr("height", indicatorHeight)
              .attr("fill", "#ff7f0e")
              .attr("clip-path", "url(#bipocClip)");
      
      // Transition the clip rectangle's width based on the bipoc rate.
      clip.transition().duration(800)
          .attr("width", bipocRate * indicatorWidth);
    }
    
    // Remove the bipoc indicator.
    function removeBipocIndicator() {
      d3.select("#bipoc").selectAll("*").remove();
    }
    
    onMount(async () => {
      // Create the main SVG for the map.
      svg = d3.select(svgElement)
              .attr("width", width)
              .attr("height", height);
      await loadData();
    });
  </script>
  
  <!-- Layout: map on left; on right, pie chart on top and bipoc indicator below -->
  <div style="display: flex; justify-content: center; align-items: flex-start;">
    <svg bind:this={svgElement} viewBox={`0 0 ${width} ${height}`}></svg>
    <div style="margin-left: 20px;">
      <div id="chart" style="width:300px; height:300px;"></div>
      <div id="bipoc" style="width:300px; height:50px; margin-top:20px;"></div>
    </div>
  </div>
  
  <style>
    svg {
      border: 1px solid #ccc;
    }
    #chart, #bipoc {
      text-align: center;
    }
    path.neighborhood {
      cursor: pointer;
    }
  </style>
  