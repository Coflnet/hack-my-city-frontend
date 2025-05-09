<template>
  <div class="min-h-screen" style="background-color: #232323;">
    <header style="background-color: #63acb8;" class="shadow-lg">
      <div class="container mx-auto px-6 py-5">
        <h1 class="text-3xl font-bold text-white">Event Explorer</h1>
      </div>
    </header>

    <main class="container mx-auto px-6 py-8">
      <div class="grid grid-cols-1 lg:grid-cols-4 gap-6">
        <!-- Filters Section -->
        <div class="lg:col-span-1 space-y-6">
          <div class="bg-white/10 backdrop-blur-sm p-5 rounded-xl shadow-lg border border-white/20">
            <h2 class="text-lg font-semibold mb-4 text-white">Event Type</h2>
            <div class="space-y-2">
              <div v-for="(type, index) in eventTypes" :key="index" class="flex items-center">
                <input type="checkbox" :id="`type-${index}`" v-model="selectedEventTypes" :value="type.value"
                  class="h-4 w-4" style="accent-color: #63acb8;"
                  @change="filterEvents" />
                <label :for="`type-${index}`" class="ml-2 text-white/90">
                  {{ type.label }}
                </label>
              </div>
            </div>
          </div>

          <div class="bg-white/10 backdrop-blur-sm p-5 rounded-xl shadow-lg border border-white/20">
            <h2 class="text-lg font-semibold mb-4 text-white">Time Period</h2>
            <div class="space-y-4">
              <div class="flex flex-col space-y-2">
                <label for="time-filter" class="text-white/90">Select view:</label>
                <select id="time-filter" v-model="selectedTimePeriod"
                  class="mt-1 block w-full pl-3 pr-10 py-2 text-base bg-white/20 border-white/30 text-white focus:outline-none focus:ring-2 focus:ring-[#63acb8] focus:border-[#63acb8] sm:text-sm rounded-lg"
                  @change="filterEvents">
                  <option value="day">Day</option>
                  <option value="week">Week</option>
                  <option value="month">Month</option>
                  <option value="year">year</option>
                </select>
              </div>

              <div v-if="selectedTimePeriod === 'day'">
                <label for="date-picker" class="text-white/90">Select date:</label>
                <input type="date" id="date-picker" v-model="selectedDate"
                  class="mt-1 block w-full bg-white/20 border-white/30 text-white rounded-lg shadow-sm focus:ring-2 focus:ring-[#63acb8] focus:border-[#63acb8] sm:text-sm"
                  @change="filterEvents" />
              </div>

              <div v-else-if="selectedTimePeriod === 'week'">
                <label for="week-picker" class="text-white/90">Select week:</label>
                <input type="week" id="week-picker" v-model="selectedWeek"
                  class="mt-1 block w-full bg-white/20 border-white/30 text-white rounded-lg shadow-sm focus:ring-2 focus:ring-[#63acb8] focus:border-[#63acb8] sm:text-sm"
                  @change="filterEvents" />
              </div>

              <div v-else-if="selectedTimePeriod === 'month'">
                <label for="month-picker" class="text-white/90">Select month:</label>
                <input type="month" id="month-picker" v-model="selectedMonth"
                  class="mt-1 block w-full bg-white/20 border-white/30 text-white rounded-lg shadow-sm focus:ring-2 focus:ring-[#63acb8] focus:border-[#63acb8] sm:text-sm"
                  @change="filterEvents" />
              </div>

              <div v-else-if="selectedTimePeriod === 'year'">
                <label for="year-picker" class="text-white/90">Select year:</label>
                <input type="number" id="year-picker" v-model="selectedYear"
                  class="mt-1 block w-full bg-white/20 border-white/30 text-white rounded-lg shadow-sm focus:ring-2 focus:ring-[#63acb8] focus:border-[#63acb8] sm:text-sm"
                  @change="filterEvents" />
              </div>
            </div>
          </div>

          <div class="bg-white/10 backdrop-blur-sm p-5 rounded-xl shadow-lg border border-white/20">
            <h2 class="text-lg font-semibold mb-4 text-white">Events Found</h2>
            <p class="text-white/90">{{ filteredEvents.length }} events match your filters</p>
          </div>
        </div>

        <!-- Map Section -->
        <div class="lg:col-span-3">
          <div class="bg-white/10 backdrop-blur-sm p-5 rounded-xl shadow-lg border border-white/20">
            <div id="map-container" class="h-[700px] w-full rounded-xl relative overflow-hidden">
              <div v-if="!mapLoaded" class="absolute inset-0 flex items-center justify-center bg-white/10 backdrop-blur-sm rounded-xl">
                <p class="text-white/70">Loading map...</p>
              </div>
              <div id="map" class="h-full w-full rounded-xl z-10"></div>
            </div>
          </div>
        </div>
      </div>
    </main>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue';

// State for filters and map
const selectedEventTypes = ref([]);
const selectedTimePeriod = ref('week');
const selectedDate = ref(new Date().toISOString().split('T')[0]);
const selectedWeek = ref(getCurrentWeek());
const selectedMonth = ref(getCurrentMonth());
const selectedYear = ref(getCurrentMonth());

const latitudeUser = ref(0);
const longitudeUser = ref(0);
const positioning = ref(true);

const events = ref([]);
const filteredEvents = ref([]);
const mapLoaded = ref(false);
let map = null;
let markers = [];
let L = null; // Store Leaflet instance

// Event types for filter
const eventTypes = [
  { label: 'News', value: 'news' },
  { label: 'Infrastruktur', value: 'infrastructure' },
  { label: 'Events', value: 'event' },
];

// Helper functions for date formatting
function getCurrentWeek() {
  const now = new Date();
  const year = now.getFullYear();
  const onejan = new Date(year, 0, 1);
  const weekNum = Math.ceil((((now - onejan) / 86400000) + onejan.getDay() + 1) / 7);
  return `${year}-W${weekNum.toString().padStart(2, '0')}`;
}

function getCurrentMonth() {
  const now = new Date();
  return `${now.getFullYear()}-${(now.getMonth() + 1).toString().padStart(2, '0')}`;
}

// Initialize Leaflet map
async function initMap() {
  try {
    // Only import Leaflet on client-side
    if (process.client) {
      // Dynamic import for Leaflet
      const leaflet = await import('leaflet');
      L = leaflet.default;

      // Make sure the map container exists
      const mapContainer = document.getElementById('map');
      if (!mapContainer) {
        console.error('Map container not found');
        return;
      }

      // Fix for marker icons in Leaflet with webpack
      delete L.Icon.Default.prototype._getIconUrl;
      L.Icon.Default.mergeOptions({
        iconRetinaUrl: 'https://unpkg.com/leaflet@1.7.1/dist/images/marker-icon-2x.png',
        iconUrl: 'https://unpkg.com/leaflet@1.7.1/dist/images/marker-icon.png',
        shadowUrl: 'https://unpkg.com/leaflet@1.7.1/dist/images/marker-shadow.png',
      });

      // Create map
      map = L.map('map').setView([48.669102, 12.690720], 13);

      L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
        maxZoom: 19,
      }).addTo(map);

      // Update markers
      updateMapMarkers();

      // Set map as loaded
      mapLoaded.value = true;

      // Force a resize to ensure the map renders correctly
      setTimeout(() => {
        if (map) map.invalidateSize();
      }, 200);
    }
  } catch (error) {
    console.error('Error initializing map:', error);
  }
}

// Update map markers based on filtered events
function updateMapMarkers() {
  if (!map || !L) return;

  // Clear existing markers
  if (markers.length) {
    markers.forEach(marker => map.removeLayer(marker));
    markers = [];
  }

  // Add new markers for filtered events
  filteredEvents.value.forEach(event => {
    const marker = L.marker([event.latitude, event.longitude])
      .addTo(map)
      .bindPopup(`
        <div>
          <h3 class="font-bold">${event.name}</h3>
          <p>${event.description}</p>
          <p>${event.category}</p>
          <p>${new Date(event.start_time).toLocaleDateString()}</p>
        </div>
      `);
    markers.push(marker);
  });

  // Adjust map view to fit all markers if there are any
  if (markers.length > 0) {
    const group = L.featureGroup(markers);
    map.fitBounds(group.getBounds().pad(0.1));
  }
}

// Fetch events - using mock data
async function fetchEvents() {
  getLocation()

  while (positioning.value) {
    await new Promise(resolve => setTimeout(resolve, 100));
  }

  events.value = await getAllEvents();
  filterEvents();
}

// Filter events based on selected filters
function filterEvents() {
  filteredEvents.value = events.value.filter(event => {
    // Filter by event type
    const typeMatch = selectedEventTypes.value.length === 0 ||
      selectedEventTypes.value.includes(event.category);

    if (!typeMatch) return false;

    // Filter by location
    if (event.latitude < 48 || event.latitude > 49 ||
      event.longitude < 11 || event.longitude > 14) {
      return false;
    }

    // Filter by time period
    let timeMatch = true;
    const eventDate = new Date(event.start_time);

    if (selectedTimePeriod.value === 'day') {
      const filterDate = new Date(selectedDate.value);
      timeMatch = eventDate.toDateString() === filterDate.toDateString();
    }
    else if (selectedTimePeriod.value === 'week') {
      const [year, week] = selectedWeek.value.split('-W');
      const firstDayOfWeek = getFirstDayOfWeek(parseInt(year), parseInt(week));
      const lastDayOfWeek = new Date(firstDayOfWeek);
      lastDayOfWeek.setDate(lastDayOfWeek.getDate() + 6);

      timeMatch = eventDate >= firstDayOfWeek && eventDate <= lastDayOfWeek;
    }
    else if (selectedTimePeriod.value === 'month') {
      const [year, month] = selectedMonth.value.split('-');
      timeMatch = eventDate.getFullYear() === parseInt(year) &&
        eventDate.getMonth() === parseInt(month) - 1;
    }
    else if (selectedTimePeriod.value === 'year') {
      const filterYear = parseInt(selectedYear.value);
      timeMatch = eventDate.getFullYear() === filterYear;
    }

    return typeMatch && timeMatch;
  });


  console.log("Filtered Events:", filteredEvents.value.length);

  // cap the filtered events to 100
  filteredEvents.value = filteredEvents.value.slice(0, 100);

  // Update map markers if map is initialized
  if (map && L) {
    updateMapMarkers();
  }
}

// Helper function to get the first day of a week
function getFirstDayOfWeek(year, week) {
  const firstDayOfYear = new Date(year, 0, 1);
  const dayOffset = firstDayOfYear.getDay() || 7;
  const dayToAdd = week * 7 - (dayOffset - 1);
  const firstDayOfWeek = new Date(year, 0, 1 + dayToAdd - 7);
  return firstDayOfWeek;
}

// Mock events for demonstration
async function getAllEvents() {
  const currentYear = new Date().getFullYear();
  const currentMonth = new Date().getMonth() + 1;

  const url = 'https://hmc.coflnet.com/api/events?lat=' + latitudeUser.value + '&lon=' + longitudeUser.value;
  const response = await fetch(url);
  if (!response.ok) {
    console.error('Failed to fetch events:', response.statusText);
    return [];
  }

  const events = await response.json();
  console.log(events)
  return events;

  // return [
  //   {
  //     id: 1,
  //     title: 'Tech Conference 2023',
  //     type: 'conference',
  //     date: `${currentYear}-${currentMonth.toString().padStart(2, '0')}-15`,
  //     lat: 48.669102,
  //     lng: 12.690720
  //   },
  //   {
  //     id: 2,
  //     title: 'JavaScript Workshop',
  //     type: 'workshop',
  //     date: `${currentYear}-${currentMonth.toString().padStart(2, '0')}-18`,
  //     lat: 48.667102,
  //     lng: 12.690720
  //   },
  //   {
  //     id: 3,
  //     title: 'Local Developers Meetup',
  //     type: 'meetup',
  //     date: `${currentYear}-${currentMonth.toString().padStart(2, '0')}-20`,
  //     lat: 48.667102,
  //     lng: 12.695720
  //   },
  //   {
  //     id: 4,
  //     title: 'Summer Music Festival',
  //     type: 'concert',
  //     date: `${currentYear}-${(currentMonth + 1).toString().padStart(2, '0')}-10`,
  //     lat: 48.667102,
  //     lng: 12.695920
  //   },
  //   {
  //     id: 5,
  //     title: 'Modern Art Exhibition',
  //     type: 'exhibition',
  //     date: `${currentYear}-${(currentMonth + 1).toString().padStart(2, '0')}-15`,
  //     lat: 48.667102,
  //     lng: 12.695420
  //   },
  //   {
  //     id: 6,
  //     title: 'Web Development Workshop',
  //     type: 'workshop',
  //     date: `${currentYear}-${(currentMonth + 1).toString().padStart(2, '0')}-25`,
  //     lat: 48.667102,
  //     lng: 12.694420
  //   }
  // ];
}


function getLocation() {
  if (!('geolocation' in navigator)) {
    this.error = 'Geolocation is not available in your browser.';
    return;
  }

  // Request permission and get position
  navigator.geolocation.getCurrentPosition(
    (position) => {
      const coords = {
        latitude: position.coords.latitude,
        longitude: position.coords.longitude,
      };
      latitudeUser.value = coords.latitude;
      longitudeUser.value = coords.longitude;
      positioning.value = false;
    },
    (error) => {
      positioning.value = false;
      switch (error.code) {
        case error.PERMISSION_DENIED:
          alert('User denied the request for Geolocation.');
          break;
        case error.POSITION_UNAVAILABLE:
          alert('Location information is unavailable.');
          break;
        case error.TIMEOUT:
          alert('The request to get user location timed out.');
          break;
        case error.UNKNOWN_ERROR:
          alert('An unknown error occurred.');
          break;
      }
    }
  );
}

// Initialize the component
onMounted(() => {
  // Set default selected event types (all selected)
  selectedEventTypes.value = [];
  for (const type of eventTypes) {
    selectedEventTypes.value.push(type.value);
    break;
  }

  // Fetch events first
  fetchEvents().then(() => {
    // Initialize map after a short delay to ensure DOM is ready
    setTimeout(() => {
      initMap();
    }, 500);
  });
});

// Clean up on component unmount
onBeforeUnmount(() => {
  if (map) {
    map.remove();
    map = null;
  }
});
</script>

<style>
/* Import Leaflet CSS */
@import 'leaflet/dist/leaflet.css';

/* Additional styles for Leaflet map */
.leaflet-container {
  font-family: inherit;
  z-index: 1;
}

.leaflet-popup-content {
  padding: 0.5rem;
}

.leaflet-popup-content h3 {
  margin-bottom: 0.25rem;
}

/* Ensure the map container has proper dimensions */
#map,
#map-container {
  min-height: 600px;
  width: 100%;
}
</style>
