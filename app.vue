<template>
  <div class="min-h-screen bg-gray-50">
    <header class="bg-white shadow-sm">
      <div class="container mx-auto px-4 py-4">
        <h1 class="text-2xl font-bold text-gray-800">Event Explorer</h1>
      </div>
    </header>

    <main class="container mx-auto px-4 py-6">
      <div class="grid grid-cols-1 lg:grid-cols-4 gap-6">
        <!-- Filters Section -->
        <div class="lg:col-span-1 space-y-6">
          <div class="bg-white p-4 rounded-lg shadow-sm">
            <h2 class="text-lg font-semibold mb-4 text-gray-800">Event Type</h2>
            <div class="space-y-2">
              <div v-for="(type, index) in eventTypes" :key="index" class="flex items-center">
                <input type="checkbox" :id="`type-${index}`" v-model="selectedEventTypes" :value="type.value"
                  class="h-4 w-4 text-emerald-600 focus:ring-emerald-500 border-gray-300 rounded"
                  @change="filterEvents" />
                <label :for="`type-${index}`" class="ml-2 text-gray-700">
                  {{ type.label }}
                </label>
              </div>
            </div>
          </div>

          <div class="bg-white p-4 rounded-lg shadow-sm">
            <h2 class="text-lg font-semibold mb-4 text-gray-800">Time Period</h2>
            <div class="space-y-4">
              <div class="flex flex-col space-y-2">
                <label for="time-filter" class="text-gray-700">Select view:</label>
                <select id="time-filter" v-model="selectedTimePeriod"
                  class="mt-1 block w-full pl-3 pr-10 py-2 text-base border-gray-300 focus:outline-none focus:ring-emerald-500 focus:border-emerald-500 sm:text-sm rounded-md"
                  @change="filterEvents">
                  <option value="day">Day</option>
                  <option value="week">Week</option>
                  <option value="month">Month</option>
                </select>
              </div>

              <div v-if="selectedTimePeriod === 'day'">
                <label for="date-picker" class="text-gray-700">Select date:</label>
                <input type="date" id="date-picker" v-model="selectedDate"
                  class="mt-1 block w-full border-gray-300 rounded-md shadow-sm focus:ring-emerald-500 focus:border-emerald-500 sm:text-sm"
                  @change="filterEvents" />
              </div>

              <div v-else-if="selectedTimePeriod === 'week'">
                <label for="week-picker" class="text-gray-700">Select week:</label>
                <input type="week" id="week-picker" v-model="selectedWeek"
                  class="mt-1 block w-full border-gray-300 rounded-md shadow-sm focus:ring-emerald-500 focus:border-emerald-500 sm:text-sm"
                  @change="filterEvents" />
              </div>

              <div v-else>
                <label for="month-picker" class="text-gray-700">Select month:</label>
                <input type="month" id="month-picker" v-model="selectedMonth"
                  class="mt-1 block w-full border-gray-300 rounded-md shadow-sm focus:ring-emerald-500 focus:border-emerald-500 sm:text-sm"
                  @change="filterEvents" />
              </div>
            </div>
          </div>

          <div class="bg-white p-4 rounded-lg shadow-sm">
            <h2 class="text-lg font-semibold mb-4 text-gray-800">Events Found</h2>
            <p class="text-gray-700">{{ filteredEvents.length }} events match your filters</p>
          </div>
        </div>

        <!-- Map Section -->
        <div class="lg:col-span-3">
          <div class="bg-white p-4 rounded-lg shadow-sm">
            <div id="map-container" class="h-[600px] w-full rounded-lg relative">
              <div v-if="!mapLoaded" class="absolute inset-0 flex items-center justify-center bg-gray-100 rounded-lg">
                <p class="text-gray-500">Loading map...</p>
              </div>
              <div id="map" class="h-full w-full rounded-lg z-10"></div>
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
const events = ref([]);
const filteredEvents = ref([]);
const mapLoaded = ref(false);
let map = null;
let markers = [];
let L = null; // Store Leaflet instance

// Event types for filter
const eventTypes = [
  { label: 'Conference', value: 'conference' },
  { label: 'Workshop', value: 'workshop' },
  { label: 'Meetup', value: 'meetup' },
  { label: 'Concert', value: 'concert' },
  { label: 'Exhibition', value: 'exhibition' }
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
  events.value = await getAllEvents();
  // filterEvents();
}

// Filter events based on selected filters
function filterEvents() {
  filteredEvents.value = JSON.parse(JSON.stringify(events.value))
  console.log(filteredEvents.value)
  if (map && L) {
    updateMapMarkers();
  }
  return;

  filteredEvents.value = events.value.filter(event => {
    // Filter by event type
    const typeMatch = selectedEventTypes.value.length === 0 ||
      selectedEventTypes.value.includes(event.type);

    // Filter by time period
    let timeMatch = true;
    const eventDate = new Date(event.date);

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

    return typeMatch && timeMatch;
  });

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

  const url = 'https://hmc.coflnet.com/api/events';
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

// Initialize the component
onMounted(() => {
  // Set default selected event types (all selected)
  selectedEventTypes.value = eventTypes.map(type => type.value);

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
