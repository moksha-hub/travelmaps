<template>
    
    <div class="min-h-screen bg-gradient-to-br from-teal-400 to-indigo-600 text-white p-4 md:p-8 overflow-hidden">
        <h1 @click="toggleModal" class="text-4xl md:text-5xl font-bold mb-6 text-center animate-fade-in-down cursor-pointer">Bali Attractions</h1>
        <div :class="{ 'hidden': !showModal }" class="fixed inset-0 flex items-center justify-center z-50">
            <div class="absolute inset-0 bg-black opacity-50" @click="toggleModal"></div>
            <div class="bg-white bg-opacity-20 backdrop-blur-md p-8 rounded-lg shadow-lg z-10 w-96">
                <h2 class="text-2xl font-bold mb-4">Set New Location</h2>
                <input
                    ref="autocompleteInput"
                    type="text"
                    placeholder="Enter a location"
                    class="input input-bordered w-full mb-4 bg-white bg-opacity-50 text-black"
                    @keyup.enter="setNewLocation">
                <button @click="setNewLocation" class="btn btn-primary w-full text-white"> Set Location </button>
            </div>
        </div>
        <div class="mb-8 bg-white bg-opacity-20 rounded-lg p-4 backdrop-blur-md">
            <label for="minRatingsCount" class="block text-lg font-medium mb-2">Minimum Ratings Count:</label>
            <div class="flex items-center">
                <input type="range" id="minRatingsCount" v-model="minRatingsCount" min="0" max="1000" step="10"
                    class="w-full mr-4 accent-teal-500">
                <span class="text-xl font-semibold">{{ minRatingsCount }}</span>
            </div>
        </div>
        <div class="md:hidden">
            <button @click="ActiveView('list')" :class="{ 'bg-teal-500': activeView === 'list' }"
                class="px-4 py-2 rounded-l-lg">
                <i class="material-icons">list</i>
            </button>
            <button @click="ActiveView('map')" :class="{ 'bg-teal-500': activeView === 'map' }"
                class="px-4 py-2 rounded-r-lg">
                <i class="material-icons">map</i>
            </button>
        </div>
        <div class="md:flex">
            <div :class="{ 'hidden md:block': activeView === 'map' }" class="md:w-1/2 md:pr-4">
                <div class="mb-4 flex justify-between items-center">
                    <div>
                        <label for="sortOrder" class="mr-2 text-white">Sort by:</label>
                        <select id="sortOrder" v-model="sortOrder" class="bg-white bg-opacity-20 rounded-lg px-2 py-1 text-white">
                            <option value="relativity" class="text-black">Relativity</option>
                            <option value="popularity" class="text-black">Popularity</option>
                            <option value="distance" class="text-black">Distance</option>
                        </select>
                    </div>
                    <button v-if="hasItineraryItems" @click="toggleItinerary" class="btn btn-primary btn-sm text-white glass">
                        <i class="material-icons mr-2 text-xl">{{ showItinerary ? 'view_list' : 'directions' }}</i>
                        <span>{{ showItinerary ? 'Show All' : 'Show Itinerary' }}</span>
                    </button>
                </div>
                <div class="grid grid-cols-1 sm:grid-cols-2 gap-6" id="attraction-grid">
                    <TransitionGroup name="attraction-list">
                        <div v-for="attraction in sortedAttractions" :key="attraction.name"
                            class="bg-white bg-opacity-10 rounded-lg overflow-hidden shadow-lg transform hover:scale-105 transition-transform duration-300 cursor-pointer"
                            @click="focusAttraction(attraction)">
                            <img :src="attraction.photoUrl" class="w-full h-48 object-cover" alt="Attraction photo">
                            <div class="p-4">
                                <h3 class="text-xl font-bold mb-2">{{ attraction.name }}</h3>
                                <p class="text-sm mb-1">
                                    <i class="material-icons text-yellow-400 align-middle mr-1">star</i> {{ attraction.rating }} / 5
                                </p>
                                <p class="text-sm mb-1">
                                    <i class="material-icons align-middle mr-1">people</i> {{ attraction.user_ratings_total }}
                                </p>
                                <p class="text-sm mb-1">
                                    <i class="material-icons align-middle mr-1">location_on</i> {{ attraction.address }}
                                </p>
                                <p class="text-sm mb-1">
                                    <i class="material-icons align-middle mr-1">directions_car</i> {{ attraction.distance.toFixed(2) }} km
                                </p>
                                <button @click.stop="toggleAttractionInItinerary(attraction)"
                                    :class="{ 'bg-green-500': attraction.inItinerary, 'bg-blue-500': !attraction.inItinerary }"
                                    class="mt-2 px-3 py-1 rounded-full text-white text-sm">
                                    <i class="material-icons align-middle mr-1">{{ attraction.inItinerary ? 'remove_circle' : 'add_circle' }}</i> {{ attraction.inItinerary ? 'Remove from Itinerary' : 'Add to Itinerary' }} </button>
                            </div>
                        </div>
                    </TransitionGroup>
                </div>
            </div>
            <div :class="{ 'hidden md:block': activeView === 'list' }" class="md:w-1/2 md:pl-4">
                <div id="map" class="h-[calc(100vh-200px)] rounded-lg shadow-lg"></div>
            </div>
        </div>
    </div>
</template>
<style scoped>
@import './styles.css';

.attraction-list-enter-active,
.attraction-list-leave-active {
  transition: all 0.5s ease;
}
.attraction-list-enter-from,
.attraction-list-leave-to {
  opacity: 0;
  transform: translateY(30px);
}

</style>
<script setup>
import { ref, onMounted, computed, watch } from 'vue';
import { getCurrentInstance } from 'vue';

const attractions = ref([]);
const minRatingsCount = ref(0);
const activeView = ref('list');
const showItinerary = ref(false);
const sortOrder = ref('distance');
let map, infowindow, markers = [], directionsService, directionsRenderer;
const homeLocation = ref({ lat: -8.4095, lng: 115.1889 });

const showModal = ref(false);
const autocompleteInput = ref(null);
let autocomplete = null;

function toggleModal() {
  showModal.value = !showModal.value;
  if (showModal.value) {
    nextTick(() => {
      autocompleteInput.value.focus();
    });
  }
}

function setNewLocation() {
  if (autocomplete) {
    const place = autocomplete.getPlace();
    if (place && place.geometry) {
      homeLocation.value = {
        lat: place.geometry.location.lat(),
        lng: place.geometry.location.lng()
      };
      map.setCenter(homeLocation.value);
      clearMarkers();
      toggleModal();
      
      // Update the title with the new location name
      if (place.name) {
        document.title = `Attractions in ${place.name}`;
        document.querySelector('h1').textContent = `${place.name} Attractions`;

      }
    }
  }
}
function clearMarkers() {
  markers.forEach(marker => marker.setMap(null));
  markers = [];
  attractions.value = [];
  directionsRenderer.setDirections({ routes: [] });
}

const filteredAttractions = computed(() => {
    return attractions.value.filter(attraction => attraction.user_ratings_total >= minRatingsCount.value);
});

const displayedAttractions = computed(() => 
    filteredAttractions.value.filter(attraction => 
        showItinerary.value === attraction.inItinerary
    )
);

const hasItineraryItems = computed(() => {
    return attractions.value.some(attraction => attraction.inItinerary);
});

const sortedAttractions = computed(() => {
    return [...displayedAttractions.value].sort((a, b) => {
        if (sortOrder.value === 'relativity') {
            return 0; // Maintain initial order
        } else if (sortOrder.value === 'popularity') {
            return b.user_ratings_total - a.user_ratings_total;
        } else {
            return a.distance - b.distance;
        }
    });
});

globalThis.initMap = function initMap() {
    
    nextTick(() => {
        autocomplete = new google.maps.places.Autocomplete(
            autocompleteInput.value,
            { types: ['(cities)'] }
        );
    });
    map = new google.maps.Map(document.getElementById('map'), {
        center: homeLocation.value,
        zoom: 12,
    });

    infowindow = new google.maps.InfoWindow();
    directionsService = new google.maps.DirectionsService();
    directionsRenderer = new google.maps.DirectionsRenderer();
    directionsRenderer.setMap(map);

    // Add listener for map idle event
    map.addListener('idle', () => {
        updateAttractions();
    });

    const homeIcon = {
        url: 'https://maps.google.com/mapfiles/kml/shapes/homegardenbusiness.png',
        scaledSize: new google.maps.Size(40, 40),
        origin: new google.maps.Point(0, 0),
        anchor: new google.maps.Point(20, 40)
    };

    new google.maps.Marker({
        position: homeLocation.value,
        map: map,
        icon: homeIcon,
        title: 'Bali'
    });
}

function updateAttractions() {
    const bounds = map.getBounds();
    const center = map.getCenter();

    const service = new google.maps.places.PlacesService(map);

    service.nearbySearch({
        location: center,
        radius: 20000,
        type: ['tourist_attraction']
    }, (results, status) => {
        if (status === google.maps.places.PlacesServiceStatus.OK) {
            results.forEach(place => {
                if (!attractions.value.some(a => a.name === place.name)) {
                    service.getDetails({ placeId: place.place_id }, (details, status) => {
                        if (status === google.maps.places.PlacesServiceStatus.OK) {
                            const attraction = {
                                name: details.name,
                                rating: details.rating,
                                user_ratings_total: details.user_ratings_total,
                                address: details.vicinity,
                                photoUrl: details.photos ? details.photos[0].getUrl({ maxWidth: 500, maxHeight: 500 }) : '',
                                location: details.geometry.location,
                                inItinerary: false,
                                distance: calculateDistance(homeLocation.value, details.geometry.location)
                            };
                            attractions.value.push(attraction);
                            createMarker(attraction);
                        }
                    });
                }
            });
        }
    });
}

function calculateDistance(point1, point2) {
    return google.maps.geometry.spherical.computeDistanceBetween(
        new google.maps.LatLng(point1),
        new google.maps.LatLng(point2)
    ) / 1000; // Convert to kilometers
}

function updateDistances() {
    const itineraryAttractions = attractions.value.filter(a => a.inItinerary);
    attractions.value.forEach(attraction => {
        if (attraction.inItinerary) {
            attraction.distance = 0; // Distance to itself is 0
        } else {
            let minDistance = calculateDistance(homeLocation.value, attraction.location);
            itineraryAttractions.forEach(itineraryAttraction => {
                const distanceToItinerary = calculateDistance(itineraryAttraction.location, attraction.location);
                if (distanceToItinerary < minDistance) {
                    minDistance = distanceToItinerary;
                }
            });
            attraction.distance = minDistance;
        }
    });
}

function createMarker(attraction) {
    const marker = new google.maps.Marker({
        map: map,
        position: attraction.location,
        title: attraction.name,
        animation: google.maps.Animation.DROP,
        icon: {
            path: google.maps.SymbolPath.CIRCLE,
            scale: 8,
            fillColor: attraction.inItinerary ? '#4CAF50' : '#FFA500',
            fillOpacity: 0.9,
            strokeWeight: 0
        }
    });

    markers.push(marker);

    marker.addListener('click', function () {
        focusAttraction(attraction);
    });
}

function ActiveView(view) {
    activeView.value = view;
    if (view === 'map') {
        calculateDirections();
    }
}

function focusAttraction(attraction) {
    activeView.value = 'map';
    map.panTo(attraction.location);
    //map.setZoom(15);
}

function toggleAttractionInItinerary(attraction) {
    attraction.inItinerary = !attraction.inItinerary;
    map.panTo(attraction.location);
    const marker = markers.find(marker => marker.getPosition().equals(attraction.location));
    marker.setIcon({
        path: google.maps.SymbolPath.CIRCLE,
        scale: 8,
        fillColor: attraction.inItinerary ? '#4CAF50' : '#FFA500',
        fillOpacity: 0.9,
        strokeWeight: 0
    });
    updateDistances();
    if(!attraction.inItinerary)
        calculateDirections();
    if (!attractions.value.some(a => a.inItinerary)) {
        activeView.value = 'map';
    }
}

function toggleItinerary() {
    showItinerary.value = !showItinerary.value;
    if (showItinerary.value) 
        calculateDirections();
    updateDistances();
}

function calculateDirections() {
    const itineraryAttractions = attractions.value.filter(a => a.inItinerary);
    if (itineraryAttractions.length < 2) return;

    const origin = itineraryAttractions[0].location;
    const destination = itineraryAttractions[itineraryAttractions.length - 1].location;
    const waypoints = itineraryAttractions.slice(1, -1).map(a => ({
        location: a.location,
        stopover: true
    }));

    directionsService.route({
        origin: origin,
        destination: destination,
        waypoints: waypoints,
        optimizeWaypoints: true,
        travelMode: 'DRIVING'
    }, (result, status) => {
        if (status === 'OK') {
            directionsRenderer.setDirections(result);
        }
    });
}


onMounted(() => {
    globalThis.vue = getCurrentInstance().setupState;
    if (window.google) {
        initMap();
    }
});

</script>
