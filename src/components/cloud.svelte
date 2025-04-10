<script lang="ts">
    import { onMount } from 'svelte';
    import * as d3 from 'd3';
  
    let svgContainer;
  
    function drawCloud(svg, x, y, scale = 1) {
      // Basic cloud shape using arcs and circles
      const cloud = d3.path();
      cloud.moveTo(x, y);
      cloud.arc(x + 20 * scale, y, 20 * scale, Math.PI, 0);
      cloud.arc(x + 50 * scale, y - 10 * scale, 25 * scale, Math.PI, 0);
      cloud.arc(x + 80 * scale, y, 20 * scale, Math.PI, 0);
      cloud.closePath();
  
      return cloud.toString();
    }
  
    onMount(() => {
      const svg = d3.select(svgContainer)
        .attr('width', 800)
        .attr('height', 200);
  
      const clouds = d3.range(3).map(i => ({
        x: i * 250,
        y: 100,
        scale: 1 + Math.random() * 0.5,
        speed: 0.1 + Math.random() * 0.3
      }));
  
      const cloudGroups = svg.selectAll('path.cloud')
        .data(clouds)
        .join('path')
        .attr('class', 'cloud')
        .attr('fill', '#ccc')
        .attr('d', d => drawCloud(svg, d.x, d.y, d.scale));
  
      d3.timer(() => {
        clouds.forEach(d => {
          d.x += d.speed;
          if (d.x > 850) d.x = -100;
        });
  
        cloudGroups
          .data(clouds)
          .attr('d', d => drawCloud(svg, d.x, d.y, d.scale));
      });
    });
  </script>
  
  <svg bind:this={svgContainer}></svg>
  
  <style>
    svg {
      background: linear-gradient(to top, #87ceeb, #ffffff);
      display: block;
      margin: auto;
    }
  </style>
  