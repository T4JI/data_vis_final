<script>
    import { onMount } from 'svelte';
    import * as d3 from 'd3';
  
    const width = 960;
    const height = 600;
    const padding = 20;
  
    let svgElement;
    let svg;
    let geoData;
    let colorScale;
    let selectedZipcode = null;
  
    let defaultProjection;
    let projection;
    let path;
  
    // Helper function to create the default projection centered on Boston.
    function createDefaultProjection() {
      return d3.geoMercator()
        .scale(70000)
        .center([-71.0589, 42.3501])
        .translate([width / 2, height / 2]);
    }
    
    defaultProjection = createDefaultProjection();
    projection = createDefaultProjection();
    path = d3.geoPath().projection(projection);
  
    // Load the combined GeoJSON (zipcodes with neighborhood property).
    async function loadData() {
      try {
        geoData = await d3.json('data/zipcode_and_neighborhood.geojson');
        
        // Extract unique neighborhood names to create a color scale.
        const neighborhoodNames = Array.from(new Set(
          geoData.features.map(d =>
            (d.properties.Neighborhood || d.properties.neighborhood || "").trim()
          ).filter(n => n !== "")
        ));
        colorScale = d3.scaleOrdinal(d3.schemeCategory10)
          .domain(neighborhoodNames);
        
        drawZipcodes();
      } catch (error) {
        console.error('Error loading GeoJSON:', error);
      }
    }
  
    // Draw each zipcode feature.
    function drawZipcodes() {
      svg.selectAll('*').remove();
      svg.selectAll('path.zipcode')
        .data(geoData.features)
        .enter()
        .append('path')
        .attr('class', 'zipcode')
        .attr('d', path)
        .attr('fill', 'none')
        .attr('stroke', d => colorScale((d.properties.Neighborhood || d.properties.neighborhood || "").trim()))
        .attr('stroke-width', 1)
        .style('cursor', 'pointer')
        .on('click', (event, d) => {
           if (!selectedZipcode || selectedZipcode !== d) {
             zipcodeClicked(d);
           } else {
             resetZoom();
           }
        });
    }
  
    // Zoom in on a clicked zipcode.
    function zipcodeClicked(feature) {
      selectedZipcode = feature;
      const t = d3.transition().duration(800);
      console.log("Before fitExtent, scale:", projection.scale());
      // Fit the projection to the clicked feature.
      projection.fitExtent([[padding, padding], [width - padding, height - padding]], feature);
      console.log("After fitExtent, scale:", projection.scale());
      path = d3.geoPath().projection(projection);
      svg.selectAll('path.zipcode')
        .transition(t)
        .attr('d', path);
    }
  
    // Reset the projection to the full Boston view.
    function resetZoom() {
      selectedZipcode = null;
      const t = d3.transition().duration(800);
      projection = createDefaultProjection();
      path = d3.geoPath().projection(projection);
      svg.selectAll('path.zipcode')
        .transition(t)
        .attr('d', path);
    }
  
    onMount(async () => {
      svg = d3.select(svgElement)
        .attr('width', width)
        .attr('height', height);
      await loadData();
    });
  </script>
  
  <svg bind:this={svgElement} viewBox={`0 0 ${width} ${height}`}></svg>
  
  <style>
    svg {
      width: 100%;
      height: auto;
      border: 1px solid #ddd;
      display: block;
      margin: 0 auto;
    }
    path.zipcode {
      cursor: pointer;
    }
  </style>
  