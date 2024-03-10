<script>
  import { onMount } from 'svelte';
  import mapboxgl from 'mapbox-gl';

  mapboxgl.accessToken = 'pk.eyJ1IjoibmF0ZG9zYW4iLCJhIjoiY2xza3huODg4MDh1ZzJpcDVoOTJ1eWFqayJ9.hvQnPf9rwTCV4aok1j7xJA';

  let mapTokyo;
  let isRailwayVisible = false;
  let areStationsVisible = false; // Initial state
  let isChoroplethVisible = false; 
  let stationMarkers = [];

  onMount(() => {
    // Tokyo map
    mapTokyo = new mapboxgl.Map({
      container: 'map-tokyo',
      style: 'mapbox://styles/mapbox/navigation-night-v1',
      center: [139.7528, 35.6852], // 35.6852° N, 139.7528° E
      zoom: 12
    });

    const popup = new mapboxgl.Popup({
      closeButton: false,
      closeOnClick: false
    });

    // Add railways
    mapTokyo.on('load', async() => {
      mapTokyo.addSource('tokyo-railways', {
        type: 'geojson',
        data: '/N02-19_RailroadSection.geojson'
      });

      // Load Population Density Choropleth source
      mapTokyo.addSource('tokyo-population-density', {
          type: 'geojson',
          data: '/tokyo23_population.geojson'
      });

      // Load administrative boundaries
      mapTokyo.addSource('tokyo-boundaries', {
        type: 'geojson',
        data: '/tokyo23_population.geojson'
      });

      mapTokyo.addLayer({
        id: 'tokyo-railways-layer',
        type: 'line',
        source: 'tokyo-railways',
        layout: {},
        paint: {
          'line-color': '#ADD8E6',
          'line-width': 2,
          'line-opacity': isRailwayVisible ? 1 : 0
        }
      });

      mapTokyo.on('mousemove', 'tokyo-railways-layer', (e) => {
        if (e.features.length > 0) {
          const feature = e.features[0];
          popup.setLngLat(e.lngLat)
               .setText(feature.properties.路線名)
               .addTo(mapTokyo);
        }
      });

      mapTokyo.on('mouseleave', 'tokyo-railways-layer', () => {
        popup.remove();
      });

      // add administrative boundary layer
      mapTokyo.addLayer({
        id: 'tokyo-administrative-boundaries',
        type: 'line',
        source: 'tokyo-boundaries',
        layout: {},
        paint: {
          'line-color': '#FFFFFF',
          'line-width': 2,
        }
      });

      // add the population density layer
      mapTokyo.addLayer({
        id: 'tokyo-population-density',
        type: 'fill',
        source: 'tokyo-population-density',
        paint: {
          'fill-color': [
            'interpolate',
            ['linear'],
            ['get', 'population'],
            0, '#ffffff',
            100000, '#614144',
            300000, '#802A37',
            500000, '#A51C2A',
            700000, '#DE0D0D',
            1000000, '#FF9641'
          ],
          'fill-opacity': 0.5
        }
      });

      mapTokyo.on('mousemove', 'tokyo-population-density', (e) => {
        if (e.features.length > 0) {
          const feature = e.features[0];
          const population = feature.properties.population; // Assuming there's a 'population' property in your geojson data
          const wardName = feature.properties.N03_004;
          const popupText = population ? `${wardName} Population: ${population}` : 'No population data available';
          popup.setLngLat(e.lngLat)
              .setText(popupText)
              .addTo(mapTokyo);
        }
      });

      const response = await fetch('/stations.json'); 
      const stationsData = await response.json(); 

      Object.values(stationsData).flat().forEach(station => {
        const score = 1 - (station.ridership / station.population);

        const marker = new mapboxgl.Marker()
          .setLngLat(station.coordinates)
          .setPopup(new mapboxgl.Popup().setText(`${station.name}: Daily Ridership - ${station.ridership}, \n Score: ${score.toFixed(2)}`))
          .addTo(mapTokyo);

        stationMarkers.push(marker); // Store the marker reference
      });
    });
  });

  // Reactive Elements
  function toggleRailwayVisibility() {
    isRailwayVisible = !isRailwayVisible;
    mapTokyo.setPaintProperty('tokyo-railways-layer', 'line-opacity', isRailwayVisible ? 1 : 0);
  }
  function toggleStationVisibility() {
    areStationsVisible = !areStationsVisible;
    stationMarkers.forEach(marker => {
      areStationsVisible ? marker.addTo(mapTokyo) : marker.remove();
  });
  }
  // Toggle choropleth visibility
  function toggleChoroplethVisibility() {
    isChoroplethVisible = !isChoroplethVisible;
    mapTokyo.setLayoutProperty('tokyo-population-density', 'visibility', isChoroplethVisible ? 'visible' : 'none');
  }
</script>

<style>
  #map-tokyo {
    position: relative; /* Needed for absolute positioning of children */
    height: 100vh;
  }

  .toggle-button {
    position: absolute; /* Position the button over the map */
    top: 20px; /* Distance from the top of the map container */
    right: 20px; /* Distance from the right side of the map container */
    z-index: 10; /* Ensure the button is above map layers and controls */
    padding: 0.5rem 1rem;
    background-color: white; /* Button background for visibility */
    border: none; /* Optional: for style */
    cursor: pointer; /* Optional: change cursor on hover */
    border-radius: 5px; /* Optional: for rounded corners */
    box-shadow: 0 2px 4px rgba(0,0,0,0.2); /* Optional: for a subtle shadow */
  }

  .map-title {
    top: 20px; /* Adjust as needed */
    left: 20px; /* Adjust as needed */
    background-color: rgba(255, 255, 255, 0.8); /* Semi-transparent background */
    padding: 0.5rem 1rem;
    border-radius: 5px;
    box-shadow: 0 2px 4px rgba(0,0,0,0.2);
  }
  .legend {
  position: absolute;
  bottom: 20px;
  left: 20px;
  background-color: rgba(255, 255, 255, 0.8);
  border-radius: 5px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.2);
  padding: 5px;
}

.legend-item {
  display: flex;
  justify-content: space-between;
  margin-bottom: 5px;
  padding: 2px 5px;
  font-size: 12px;
}
</style>

<div id="map-tokyo">

</div>

<div class="map-title">Tokyo Live Roads with Metro Lines</div>
<button class="toggle-button" on:click={toggleRailwayVisibility}>
  {isRailwayVisible ? 'Hide Railway Lines' : 'Show Railway Lines'}
</button>
<button class="toggle-button" on:click={toggleStationVisibility} style="top: 80px;">
  {areStationsVisible ? 'Hide Stations' : 'Show Stations'}
</button>
<button class="toggle-button" on:click={toggleChoroplethVisibility} style="top: 140px;">
  {isChoroplethVisible ? 'Hide Choropleth' : 'Show Choropleth'}
</button>

<div class="legend" style="bottom: 200px;">
  <div class="legend-item" style="background-color: #ffffff;">0</div>
  <div class="legend-item" style="background-color: #614144;">100,000</div>
  <div class="legend-item" style="background-color: #802A37;">300,000</div>
  <div class="legend-item" style="background-color: #A51C2A;">500,000</div>
  <div class="legend-item" style="background-color: #DE0D0D;">700,000</div>
  <div class="legend-item" style="background-color: #FF9641;">1,000,000</div>
</div>