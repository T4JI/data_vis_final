<script>
  import Map from './Map.svelte';
  import PieChart from './PieChart.svelte';
  import RaceBarChart from './RaceBarChart.svelte';
  
  let selectedDetails = null;
  let selectedNeighborhoodName = "";
  
  function handleSelectNeighborhood(event) {
    selectedDetails = event.detail.details;
    selectedNeighborhoodName = event.detail.name;
  }
  
  function handleResetNeighborhood() {
    selectedDetails = null;
    selectedNeighborhoodName = "";
  }
</script>
<header style="text-align: center; padding: 20px;">
  <h1>Bubble Bursters Interactive Map</h1>
  <p> Click on a Neighborhood to zoom in, click again to zoom out.</p>
  
</header>
<div class="dashboard-container">
  <div style="display: flex; justify-content: center; align-items: flex-start;">
    <Map 
      on:selectNeighborhood={handleSelectNeighborhood}
      on:resetNeighborhood={handleResetNeighborhood} />
    <div style="margin-left: 20px; display: flex; flex-direction: column; align-items: center;">
      {#if selectedDetails}
        <PieChart details={selectedDetails} />
      {/if}
      <RaceBarChart selectedNeighborhoodName={selectedNeighborhoodName} />
    </div>
  </div>
  
</div>
<style>
  .dashboard-container {
    max-width: 960px;
    width: 90%;
    margin: 0 auto;
    padding: 10px;
  }
  @media (max-width: 600px) {
    .dashboard-container { padding: 5px; }
  }
</style>
