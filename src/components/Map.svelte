<script>
    import { onMount } from 'svelte';
    import * as d3 from 'd3';
  
    // Set dimensions and padding.
    const width = 960;
    const height = 600;
    const padding = 20;
  
    let svgElement;
    let svg;
  
    let geoData;
    let neighborhoods; // Array of neighborhood features.
    let selectedNeighborhood = null; // Track which neighborhood is zoomed in.
  
    // Function to create the default Boston projection.
    function createDefaultProjection() {
      return d3.geoMercator()
        .scale(70000)
        .center([-71.0589, 42.3501])
        .translate([width / 2, height / 2]);
    }
  
    // Create the initial projection.
    let projection = createDefaultProjection();
    let path = d3.geoPath().projection(projection);
  
    // Load the neighborhoods GeoJSON.
    async function loadData() {
      try {
        geoData = await d3.json('data/neighborhoods.geojson');
        neighborhoods = geoData.features;
        drawMap();
      } catch (error) {
        console.error('Error loading GeoJSON:', error);
      }
    }
  
    // Draw all neighborhood paths.
    function drawMap() {
      svg.selectAll('*').remove();
      svg.selectAll('path.neighborhood')
        .data(neighborhoods)
        .enter()
        .append('path')
        .attr('class', 'neighborhood')
        .attr('d', path)
        .attr('fill', '#ccc')
        .attr('stroke', '#333')
        .on('click', function(event, d) {
          // If no neighborhood is currently selected, or a different one is clicked,
          // zoom in on the clicked feature.
          if (selectedNeighborhood !== d) {
            selectedNeighborhood = d;
            zoomIn(d);
          } else {
            // If the same neighborhood is clicked again, reset the zoom.
            selectedNeighborhood = null;
            resetZoom();
          }
        });
    }
  
    // Zoom in on a given neighborhood feature.
    function zoomIn(feature) {
      const t = d3.transition().duration(800);
      // Update the projection to fit the clicked feature.
      projection.fitExtent([[padding, padding], [width - padding, height - padding]], feature);
      // Update the path generator.
      path = d3.geoPath().projection(projection);
      // Transition all neighborhood paths to the new projection.
      svg.selectAll('path.neighborhood')
        .transition(t)
        .attr('d', path);
    }
  
    // Reset the projection back to the default Boston view.
    function resetZoom() {
      const t = d3.transition().duration(800);
      projection = createDefaultProjection();
      path = d3.geoPath().projection(projection);
      svg.selectAll('path.neighborhood')
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
      border: 1px solid #ccc;
      display: block;
      margin: auto;
    }
    path.neighborhood {
      cursor: pointer;
    }
  </style>
  