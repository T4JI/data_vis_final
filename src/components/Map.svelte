<script>
    import { onMount } from 'svelte';
    import * as d3 from 'd3';
  
    const width = 960;
    const height = 600;
    const padding = 20;
  
    let svgElement;
    let svg;
  
    // These will hold the separate GeoJSON data.
    let neighborhoodsGeo; // Neighborhoods GeoJSON.
    let zipcodesGeo;      // Zipcodes GeoJSON.
  
    let currentView = 'neighborhood'; // Either "neighborhood" or "zipcode"
    let selectedNeighborhood = null;
  
    // Create a projection and a path generator.
    // Set initial parameters to show the full area.
    const projection = d3.geoMercator()
      .scale(70000)
      .center([-71.0589, 42.3501])
      .translate([width / 2, height / 2]);
    const path = d3.geoPath().projection(projection);
  
    // Load both GeoJSON files.
    async function loadData() {
      try {
        neighborhoodsGeo = await d3.json('data/neighborhoods.geojson');
        zipcodesGeo = await d3.json('data/zipcode_and_neighborhood.geojson');
        console.log('Neighborhoods GeoJSON loaded:', neighborhoodsGeo);
        console.log('Zipcodes GeoJSON loaded:', zipcodesGeo);
        drawNeighborhoods();
      } catch (error) {
        console.error('Error loading GeoJSON:', error);
      }
    }
  
    // Draw the neighborhood outlines.
    function drawNeighborhoods() {
      currentView = 'neighborhood';
      svg.selectAll('*').remove();
      svg.selectAll('path.neighborhood')
        .data(neighborhoodsGeo.features)
        .enter()
        .append('path')
        .attr('class', 'neighborhood')
        .attr('d', path)
        .attr('fill', '#ccc')
        .attr('stroke', '#333')
        .style('cursor', 'pointer')
        .on('click', (event, d) => {
          console.log('Neighborhood clicked:', d.properties);
          neighborhoodClicked(d);
        });
    }
  
    // When a neighborhood is clicked.
    function neighborhoodClicked(neighFeature) {
      currentView = 'zipcode';
      selectedNeighborhood = neighFeature;
  
      // Zoom into the neighborhood using projection.fitExtent.
      projection.fitExtent([[padding, padding], [width - padding, height - padding]], neighFeature);
  
      // Transition neighborhood paths to indicate selection.
      // Wait for transition to finish before drawing zipcodes.
      svg.selectAll('path.neighborhood')
        .transition().duration(800)
        .attr('d', path)
        .attr('stroke', '#888')
        .on('end', () => {
          drawZipcodes(neighFeature);
        });
    }
  
    // Draw the zipcode features for the selected neighborhood.
    function drawZipcodes(neighFeature) {
      // Get the neighborhood name from the clicked feature.
      const neighName = (neighFeature.properties.Neighborhood ||
                         neighFeature.properties.neighborhood ||
                         neighFeature.properties.name ||
                         "").trim();
      console.log('Selected Neighborhood name:', neighName);
  
      // Filter zipcode features that belong to the clicked neighborhood.
      const filteredZipcodes = zipcodesGeo.features.filter(f => {
        let fNeigh = (f.properties.Neighborhood || f.properties.neighborhood || "").trim();
        return fNeigh === neighName;
      });
      console.log('Filtered zipcode count:', filteredZipcodes.length);
  
      // Create a group for zipcode paths.
      const zipcodeGroup = svg.append('g')
        .attr('class', 'zipcode-group');
  
      // Calculate the centroid of the neighborhood.
      const centroid = path.centroid(neighFeature);
      console.log('Neighborhood centroid:', centroid);
  
      // Apply a transform on the zipcode group to scale down around the centroid.
      // Adjust scale factor as desired (e.g., 0.9 means 90% size).
      const scaleFactor = 0.99;
      zipcodeGroup.attr('transform', `translate(${centroid[0]},${centroid[1]}) scale(${scaleFactor}) translate(${-centroid[0]},${-centroid[1]})`);
  
      // Draw the filtered zipcode boundaries inside the group.
      zipcodeGroup.selectAll('path.zipcode')
        .data(filteredZipcodes)
        .enter()
        .append('path')
        .attr('class', 'zipcode')
        .attr('d', path)
        .attr('fill', 'none')
        .attr('stroke', 'red')
        .attr('stroke-width', 2);
    }
  
    // Back button handler to return to neighborhoods view.
    function goBack() {
      currentView = 'neighborhood';
      selectedNeighborhood = null;
      // Reset the projection to initial settings.
      projection
        .scale(70000)
        .center([-71.0589, 42.3501])
        .translate([width / 2, height / 2]);
      // Remove the zipcode layer.
      svg.selectAll('g.zipcode-group').remove();
      // Redraw the neighborhoods.
      drawNeighborhoods();
    }
  
    onMount(async () => {
      svg = d3.select(svgElement)
        .attr('width', width)
        .attr('height', height);
      await loadData();
    });
  </script>
  
  <div>
    {#if currentView === 'zipcode'}
      <button on:click={goBack}>Back to Neighborhoods</button>
    {/if}
  </div>
  <svg bind:this={svgElement} viewBox={`0 0 ${width} ${height}`}></svg>
  
  <style>
    svg {
      width: 100%;
      height: auto;
      border: 1px solid #ddd;
      display: block;
      margin: 0 auto;
    }
    button {
      margin: 10px;
      padding: 0.5em 1em;
      font-size: 1em;
    }
  </style>
  