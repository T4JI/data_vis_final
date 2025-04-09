<script>
  import Map from './Map.svelte';
  import PieChart from './PieChart.svelte';
  import RaceBarChart from './RaceBarChart.svelte';
  
  // These variables store details for the currently selected neighborhood.
  // For the pie chart, we need the condo conversion details.
  let selectedDetails = null;
  // For the race bar chart, we need the neighborhood name (or an empty string for default averages).
  let selectedNeighborhoodName = "";
  
  // Handle selection event from Map.svelte.
  // Expected event.detail: { name, details, bipocRate } – here we take "name" and "details."
  function handleSelectNeighborhood(event) {
    selectedDetails = event.detail.details;
    selectedNeighborhoodName = event.detail.name;
  }
  
  // Handle reset event.
  function handleResetNeighborhood() {
    // When resetting, we want our RaceBarChart to show the overall average.
    // For that, we simply pass an empty string (or you can pass a fixed "Average" value).
    selectedDetails = null;
    selectedNeighborhoodName = "";
  }
</script>

<header style="text-align: center; padding: 20px;">
  <h1>Bubble Bursters Interactive Map</h1>
  <h3>Group: Shivali Singireddy, Subhash Kantamneni, Irura Nyiha, Taji Manning</h3>
  <p>A dynamic data visualization project exploring Boston neighborhoods.</p>
</header>

<div style="display: flex; justify-content: center; align-items: flex-start;">
  <Map 
    on:selectNeighborhood={handleSelectNeighborhood}
    on:resetNeighborhood={handleResetNeighborhood}
  />
 <div style="display: flex; flex-direction: column; align-items: center;">
    {#if selectedDetails}
      <PieChart details={selectedDetails} />
    {/if}
    <div style="margin-left: 100px;">
      <RaceBarChart selectedNeighborhoodName={selectedNeighborhoodName} />
    </div>
  </div>
</div>
