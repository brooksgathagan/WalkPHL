<template>
  <div id="app" class="map-widget">
    <!-- <div id="swiper" v-touch:swipe.top="swipeUpSidebar" v-touch:swipe.bottom="swipeDownSidebar"> -->

    <!-- entire side bar -->
    <div v-show="panel" id="right-panel">
    
      <!-- back button to go back to destination choice list -->
      <button v-show="choseDestination" class="button" id="back-button" v-on:click="showDestinationDivs">&#10229; Back</button>

      <p v-if="!choseDestination"><b>Map style:</b></p>
      <div class="location-buttons select" v-if="!choseDestination">
      <select v-if="!choseDestination" id="mapmode" v-on:change="toggleMapMode" v-model="mapMode">
        <option disabled selected value>Select a map style:</option>
        <option value="default">Default</option>
        <option value="silver">Silver</option>
        <option value="retro">Retro</option>
        <option value="dark">Dark</option>
        <option value="night">Night</option>
        <option value="aubergine">Aubergine</option>
      </select>
      </div>

      

      <!-- buttons at the top -->
      <div class="location-buttons" v-show="choseDestination">
        
        <button id="dir" v-on:click="renderDirectionsButton" class="button is-primary" >Get Directions</button>
        
        <button
          class="button is-primary button-2"
          v-if="userLoggedIn"
          v-on:click="reviewButtonClicked"
        >Show Reviews</button>
        <button id="check-in-button" :disabled="checkedIn" class="button is-primary button-3" v-if="userLoggedIn" v-on:click="checkIn()">Check-In</button>   
      </div>
      <p v-show="checkedIn">Hope you enjoy your visit!</p>

      <p v-show="!choseDestination"><b>Filter locations by distance:</b></p>
      <!-- user selects radius -->
      <div class="location-buttons select" v-show="!choseDestination">
        <select v-on:change="filterIcons" v-model="radiusFilter" id="radius">
          <option disabled selected value="100000">Limit search radius to:</option>
          <option value="250">250 Meters</option>
          <option value="804">Half Mile</option>
          <option value="1607">One Mile</option>
          <option value="4821">Three Miles</option>
          <option value="16092">Ten Miles</option>
          <option value="100000000000000">I'm down to travel</option>
        </select>
      </div>
      <p v-show="!choseDestination"><b>Search locations and categories:</b></p>
      <input type="text" id="searchfilter" class="input location-buttons" placeholder="Enter a location or category" v-show="!choseDestination" v-model="searchText" v-on:input="filterIcons">

      <p class="directions-error" v-show="!selectedMode">Please select a travel mode from the drop-down menu below.</p>

      <!-- user selects transportation type -->
      <p v-show="hasSelectedTravelMode"></p>
      <div class="location-buttons select" v-show="choseDestination">
        <select id="type" v-model="currentTravelMode" v-on:change="renderDirectionsMode">
          <option disabled selected value>Select a travel mode:</option>
          <option value="WALKING">Walk PHL!</option>
          <option value="BICYCLING">Bicycle</option>
          <option value="DRIVING">Drive</option>
          <option value="TRANSIT">Public Transit</option>
        </select>
      </div>

      <!-- displays and loops through all of the locations -->
      <div id="destination-div">
        <div class="container" v-show="!choseDestination">
          <div
            class="box destination-list hover-mouse center-text"
            v-bind:id="destination.destinationId"
            v-on:click="chooseDestination(destination)"
            v-for="destination in filterSearch()"
            :key="destination.destinationId"
            v-bind:value="destination.name"
          >
            <img :src="require(`../assets/images/${destination.imgUrl}`)" alt="image of a destination in this list"/>
            <h4><b>{{destination.name}}</b></h4>
            <p><em>{{destination.category}}</em></p>
            <p style="text-align:left">{{destination.description}}</p>
            <p><b>Hours:</b> {{destination.openFrom}} - {{destination.openTo}}</p>
            <p><b>Open Weekends:</b> {{destination.openOnWeekends}}</p>
          </div>
        </div>

        <div v-if="choseDestination" class="box center-text">
          <img :src="require(`../assets/images/${currentDestination.imgUrl}`)" alt="image of current destination"/>
          <h4><b>{{currentDestination.name}}</b></h4>
          <p><em>{{currentDestination.category}}</em></p>
          <p style="text-align:left">{{currentDestination.description}}</p>
          <p><b>Hours:</b>  {{currentDestination.openFrom}} - {{currentDestination.openTo}}</p>
          <p><b>Open Weekends:</b>  {{currentDestination.openOnWeekends}}</p>
          <a class="icon" target="_blank" v-if="currentDestination.twitterHandle" v-bind:href="'https://twitter.com/' + currentDestination.twitterHandle">
            <img class="fas fa-home" :src="require(`../assets/images/twitter.png`)" alt="twitter icon"/>
          </a>    
          <!-- commented out because we don't have a fb column in the db yet with their respective urls
          do a null check if the url exists, see above twitter link for an example
          <a class="icon" v-bind:href="'https://facebook.com/' + currentDestination.facebookPage">
            <img class="fas fa-home" src="/assets/images/facebook.png" alt="facebook icon"/>
          </a>   -->     
       </div>

      </div>

      <!-- displays all reviews (not a particular location yet) -->
      <div v-if="this.displayReviews" id="review-div">
        <div class="container">
          <div class="box" v-for="review in filterReviews()" :key="review.review_id">
            <article class="media">
              <!-- placeholder for review image -->
              <div class="media-left">
                <figure class="image is-64x64">
                  <img :src="require(`../assets/images/plane.png`)" alt="user icon image">
                </figure>
              </div>

              <div class="media-content">
                <!-- review info -->
                <div class="content">
                  <p>
                    <strong>{{review.username}}</strong>
                    <br />
                    <br />
                    <strong>{{review.title}}</strong>
                    <br />
                    {{review.review}}
                  </p>
                </div>

                <!-- review date which isn't showing? -->
                <nav class="level is-mobile">
                  <div class="level-left">
                    <p class="level-item">
                      <span>{{review.review_date}}</span>
                    </p>
                  </div>
                </nav>
              </div>
            </article>
          </div>
        </div>

        <!-- user can leave a review if they're logged in -->
        <form @submit.prevent="leaveReview" v-if="userLoggedIn">
          <p>
            <strong>Leave a review?</strong>
          </p>

          <label for="title">
            <input
              type="text"
              v-model="review.title"
              placeholder="Review Title"
              id="title"
              class="input"
              maxlength="40"
              required
              autofocus
            />
          </label>

          <label for="review">
            <textarea
              type="text"
              v-model="review.review"
              placeholder="Review..."
              id="review"
              class="textarea is-primary"
              maxlength="255"
              required
              autofocus
            ></textarea>
          </label>

          <button class="button" type="submit">Submit Review</button>
        </form>
      </div>
    </div>

    <!-- displays the map -->
    <div id="map" v-bind:class="{'maps': panel}"></div>
  </div>
</template>

<script>

import gmapsInit from "./../utils/gmaps";
import auth from "../auth";
import mapstyles from "../mapstyles.json";

let google = null;
let geocoder = null;
let map = null;
let directionsService = null;
let directionsRenderer = null;

//const vm = this;

const techelevator = {
  name: '???',
  category: '',
  description: 'Like our app? Find your way here to learn how to make your own.',
  latitude: 39.949390,
  longitude: -75.169212,
  iconUrl: 'question.png',
  imgUrl: 'dmp.jpg'
};

function filterByRadius(location, pos) {
  var R = 6371000;
  var dLat = (Math.PI / 180) * (parseFloat(location.latitude) - pos.lat);
  var dLon = (Math.PI / 180) * (parseFloat(location.longitude) - pos.lng);
  var lat1 = (Math.PI / 180) * pos.lat;
  var lat2 = (Math.PI / 180) * pos.lng;
  var a =
    Math.sin(dLat / 2) * Math.sin(dLat / 2) +
    Math.sin(dLon / 2) *
      Math.sin(dLon / 2) *
      Math.cos(lat1) *
      Math.cos(lat2);
  var c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
  var d = R * c;
  return d;
}

export default {
  name: "map-widget",
  components: {},
  data() {
    return {
      searchText:'',
      choseDestination: false,
      hasPosition: false,
      showDestination: false,
      displayInfo: false,
      displayReviews: false,
      destinationPosition: {
        lat: 0,
        lng: 0
      },
      userPosition: {
        lat: 0,
        lng: 0
      },
      destinationName: "",
      review: {
        username: '',
        title: "",
        review: "",
        destinationId: "",
      },
      checkInObject: {
        username: '',
        destinationId: ''
      },
      username: '',
      reviews: [],
      destinations: [],
      radiusFilter: 100000,
      markers: [],
      currentDestination: '',
      currentTravelMode: '',
      checkedIn: false,
      selectedMode: true,
      mapMode: '',
      placeId: null,
    };
  },
  props: {
    panel: Boolean
  },
  computed: {
    userLoggedIn() {
      return auth.getToken() != null;
    },
    hasSelectedTravelMode() {
      return this.currentTravelMode === '';
    }
  },
  methods: {
    toggleMapMode() {
      if (this.mapMode === 'dark') {
        map.setOptions({styles: mapstyles.dark});
      } else if (this.mapMode === 'retro') {
        map.setOptions({styles: mapstyles.retro});
      } else if (this.mapMode === 'silver') {
        map.setOptions({styles: mapstyles.silver});
      } else if (this.mapMode === 'aubergine') {
        map.setOptions({styles: mapstyles.aubergine});
      } else if (this.mapMode === 'night') {
        map.setOptions({styles: mapstyles.night})
      } else {
        map.setOptions({styles: mapstyles.default});
      }
    },
    filterSearch() {
      
      const vm = this;
      const filter = new RegExp(this.searchText, 'i');
      return this.destinations.filter((destination) => {
        return (destination.name.match(filter) || destination.category.match(filter));
      }).filter(destination => {
        return filterByRadius(destination, vm.userPosition) <= vm.radiusFilter})
      
    },
    filterReviews() {
      return this.reviews.filter((review) => {
        return (review.destinationId == this.currentDestination.destinationId);
      })
    },
    getReviews() {
      fetch(`${process.env.VUE_APP_REMOTE_API}/recent-reviews`, {
        method: "GET",
        headers: {
          Authorization: "Bearer " + auth.getToken()
        }
      })
        .then(response => {
          if (response.ok) {
            console.log(response);
            return response.json();
          }
        })
        .then(reviews => {
          this.reviews = reviews;
        })
        .catch(err => {
          console.log(err);
        });
    },
    resetDirectionsBackButton() {
        directionsRenderer.setMap(null);
        directionsRenderer.setPanel(null);
        document.getElementById("dir").innerText = "Get Directions";
    },
    chooseDestination(destination) {
      this.choseDestination = true;
      this.currentDestination = destination;
      directionsRenderer.map = null;
      
      // Note to self: This + the callback method below stores a PlaceID. With this, we'll be able to get address/hours/pictures/etc etc. dynamically.

      var request = {
        query: this.currentDestination.name + ', Philadelphia'
      };

      var service = new google.maps.places.PlacesService(map);
      service.textSearch(request, this.callback);

    },
    callback(results, status) {
      const vm = this;
      if (status == google.maps.places.PlacesServiceStatus.OK) {
        vm.placeId =  results[0].place_id;
        console.log(vm.placeId);
      }
    },
    showDestinationDivs() {
      this.checkedIn = false;
      this.choseDestination = false;
      document.getElementById("type").value = "";
      this.displayReviews = false;
      document.getElementById("dir").innerText = "Get Directions";
      this.resetDirectionsBackButton();
    },
    swipeUpSidebar() {
      let swipeDiv = document.getElementById("swiper");
      swipeDiv.classList.add("swipeUp");
    },
    swipeDownSidebar() {
      let swipeDiv = document.getElementById("swiper");
      swipeDiv.classList.add("swipeDown");
    },
    toRad(degree) {
      return (degree * Math.PI) / 180;
    },
    leaveReview() {
      this.review.destinationId = this.currentDestination.destinationId;
      this.review.username = this.username;
      fetch(`${process.env.VUE_APP_REMOTE_API}/leave-review`, {
        method: "POST",
        headers: {
          Authorization: "Bearer " + auth.getToken(),
          Accept: "application/json",
          "Content-Type": "application/json"
        },
        body: JSON.stringify(this.review)
      })
        .then(response => {
          if (response.ok) {
            this.$router.push({ path: "/reviews" });
            console.log("success! review posted");
          } else {
            console.log("error leaving review");
          }
        })
        .then(err => console.log(err));
    },
    reviewButtonClicked() {
      this.displayReviews = !this.displayReviews;
    },
    fetchDestinations() {
      let promise = null;
        promise = fetch(`${process.env.VUE_APP_REMOTE_API}/destinations`, {
        method: "GET"
      })
        .then(response => {
          if (response.ok) {
            console.log(response);
            return response.json();
          }
        })
        .then(destinations => {
          console.log(destinations);
          this.destinations = destinations;
        })
        .catch(err => {
          console.log(err);
        });
        return promise;
    },
    checkIn() {
      this.checkedIn = true;
      this.checkInObject.username = this.username;
      this.checkInObject.destinationId = this.currentDestination.destinationId;

      fetch(`${process.env.VUE_APP_REMOTE_API}/check-in`, {
          method: "POST",
          headers: {
            Authorization: "Bearer " + auth.getToken(),
            Accept: "application/json",
            "Content-Type": "application/json"
          },
          body: JSON.stringify(this.checkInObject)
        }
      )
        .then(response => {
          if (response.ok) {
            // this.$router.push({ path: "/" });
            console.log("Successful check-in");
          } else {
            console.log("Error checking in");
          }
        })
        .then(err => console.log(err));
    },

    addMarker(location) {
      const vm = this;
      var scales = null;

       if (location.iconUrl === 'arts.png') {
        scales = new google.maps.Size(35, 35);
      }  else if (location.iconUrl === 'globe.png') {
        scales = new google.maps.Size(35, 35);
      } else {
        scales = new google.maps.Size(30, 30);
      }

      var marker = new google.maps.Marker({
        position: new google.maps.LatLng(location.latitude, location.longitude),
        map,
        title: location.name,
        draggable: false,
        icon: {url: require(`../assets/images/${location.iconUrl}`), scaledSize: scales},
      });
      let infowindow = null;
      marker.addListener(`dblclick`, () => this.markerClickHandler(marker));

      if (location.category === 'secret') {
        infowindow = new google.maps.InfoWindow({ content: '<b><h1 style="padding-bottom: 4px"></b>' + location.name + '</h1><p style="padding-bottom: 4px">' + location.description + '</p> <img src="' + require(`../assets/images/${location.imgUrl}`) + '" alt="a secret" height="150" width="150"/>'});
      } else {
        infowindow = new google.maps.InfoWindow({ content: '<b><h1 style="padding-bottom: 4px"></b>' + location.name + '</h1><p style="padding-bottom: 4px">' + location.description + '</p> <p><a style="color: blue" target="_blank" href="' + location.wiki + '">View on Wikipedia</a></p>'});
      }
      marker.addListener("click", function() {
        if (!this.displayInfo) {
          infowindow.open(map, marker);
          this.displayInfo = !this.displayInfo;
        } else {
          infowindow.close(map, marker);
          this.displayInfo = !this.displayInfo;
        }

        // Syncs up icon clicks with sidebar. Every so often, an infowindow won't display properly, but the sidebar changes. Not sure what's going on there yet, but ignoring it for now - Brooks
        if (!vm.choseDestination) {
          vm.chooseDestination(location) 
        } else if (vm.choseDestination && (vm.currentDestination != location)){
          vm.chooseDestination(location);
        } else {
          vm.choseDestination = false;
        }
      });

      this.markers.push(marker);
    },

    markerClickHandler(marker) {
      if (map.getZoom() == 17)
        map.setZoom(14);
      else
        map.setZoom(17);

      map.setCenter(marker.getPosition());
    },
    /*
    filterDestinations() {
      for (var i = 0; i < this.markers.length; i++) {  
        this.markers[i].setMap(null);
      }
      const currentUserPosition = this.userPosition;
      const currentRadiusFilter = this.radiusFilter;
      this.destinations.forEach(location => {
        if (filterByRadius(location, currentUserPosition) <= currentRadiusFilter) {
          this.addMarker(location);
        }
      })
      if (filterByRadius(techelevator, currentUserPosition) <= currentRadiusFilter) {
        this.addMarker(techelevator);
      }
    }, */
    filterIcons() {
      for (var i = 0; i < this.markers.length; i++) {  
        this.markers[i].setMap(null);
      }
      const filter = new RegExp(this.searchText, 'i');
      this.destinations.forEach(location => {
        if (location.name.match(filter) || location.category.match(filter)) {
          if (filterByRadius(location, this.userPosition) <= this.radiusFilter) {
            this.addMarker(location);
          }
          
        }
      })
      if (techelevator.name.match(filter) || techelevator.category.match(filter)) {
        if (filterByRadius(location, this.userPosition) <= this.radiusFilter) {
          this.addMarker(techelevator);
        }
        
      }
    },
    calculateAndDisplayRoute() {
      // var end = document.getElementById('end').value;
      var type = document.getElementById("type").value;

      directionsService.route(
        {
          origin: this.userPosition,
          destination: {lat: parseFloat(this.currentDestination.latitude), lng: parseFloat(this.currentDestination.longitude)},
          travelMode: type
        },
        function(response, status) {
          if (status === "OK") {
            directionsRenderer.setDirections(response);
          }
        }
      );
    },
    renderDirectionsButton() {
      if (directionsRenderer.getMap() != null) {
        directionsRenderer.setMap(null);
        directionsRenderer.setPanel(null);
        document.getElementById("dir").innerText = "Get Directions";
      } else {
        this.renderDirections();
        document.getElementById("dir").innerText = "Hide Directions";
      }
    },    
    renderDirectionsMode() {
      if (directionsRenderer.map != null) {
         this.renderDirections();
      }
    },
    renderDirections(){
        directionsRenderer.setPanel(document.getElementById("right-panel"));
        directionsRenderer.setMap(map);
        this.calculateAndDisplayRoute();
    },
  },
  created() {
    this.reviews = this.getReviews();
    if (auth.loggedIn()) {
      this.username = auth.getUser().sub;
    } 
  },
  async mounted() {
    await this.fetchDestinations();
    // Initialization of Maps objects
    google = await gmapsInit();
    geocoder = new google.maps.Geocoder();
    map = new google.maps.Map(document.getElementById("map"), {
      styles: mapstyles.default,
    });

    //const places = new google.maps.places.PlacesService(map);
    directionsService = new google.maps.DirectionsService();
    directionsRenderer = new google.maps.DirectionsRenderer();

    directionsRenderer.setPanel(document.getElementById("right-panel"));
    directionsRenderer.setMap(map);

    // Geocoder is grabbing places that match our address search. It returns an array, but given we're specific, it will only be of size one.
    // We reference our search it with results[0], and declare the map to be centered on the location of our search (Center City)
    geocoder.geocode(
      { address: `Center City, Philadelphia` },
      (results, status) => {
        if (status !== `OK` || !results[0]) {
          throw new Error(status);
        }
        map.setCenter(results[0].geometry.location);
        map.fitBounds(results[0].geometry.viewport);
      }
    );

        // Default: Center City, PHILADELPHIA
    this.userPosition.lat = 39.9509;
    this.userPosition.lng = -75.1575;

    // The next few lines requests a user's position (I was using Vue's before; this is Google's)
    if (navigator.geolocation) {
      const position = await new Promise((resolve, reject) =>
        navigator.geolocation.getCurrentPosition(resolve, reject)
      ).then(post => post);

      this.userPosition.lat = position.coords.latitude;
      this.userPosition.lng = position.coords.longitude;
    }

    // Getting an array of markets to plop on our map
    this.addMarker(techelevator);
    this.destinations.map(location => {
      this.addMarker(location);
    });

    // This sets an infowindow on our current location.
    var infoWindow = new google.maps.InfoWindow();
    infoWindow.setPosition(this.userPosition);
    infoWindow.setContent("Oh, there you are! Haiii");
    infoWindow.open(map);
    map.setCenter({ lat: 39.9526, lng: -75.1652 });
    map.setZoom(14);
    
  }
};


</script>


<style>
@import url("https://fonts.googleapis.com/css2?family=Francois+One&display=swap");

.hide-form {
  display: none;
}

#app {
  width: 100vw;
  height: 100vh;
}

.map-widget {
  padding-top: 5rem;
}

.swipeUp {
  animation-name: swipeUp;
  animation-duration: 0.5s;
}

@keyframes swipeUp {
  0% {
    bottom: -50px;
  }
  100% {
    bottom: 0;
  }
}

.swipeDown {
  animation-name: swipeDown;
  animation-duration: 0.5s;
}

@keyframes swipeDown {
  0% {
    top: 0px;
  }
  100% {
    top: 50px;
  }
}

@media only screen and (max-width: 768px) {
  .location-buttons {
    display: grid;
    grid-template-columns: 1fr;
  }

  #right-panel {
    height: 100%;
    float: right;
    width: 100%;
    overflow: auto;
  }
}

@media only screen and (min-width: 768px) {
  .location-buttons {
    display: grid;
    grid-template-columns: 3fr;
    grid-gap: 1em;
  }

  .button-1 {
    grid-column: 1 / 1;
    padding: 1em;
  }

  .button-2 {
    grid-column: 2 / 2;
    padding: 1em;
  }

  .button-3 {
    grid-column: 3 / 3;
    padding: 1em;
  }

  #right-panel {
    height: 100%;
    float: right;
    width: 25%;
    overflow: auto;
  }
}

/* Always set the map height explicitly to define the size of the div
  * element that contains the map.*/
#map {
  height: 100%;
  padding-top: 5rem;
}

/* Optional: Makes the sample page fill the window. */
html,
body {
  height: 100%;
  margin: 0;
  padding: 0;
}
#floating-panel {
  position: absolute;
  top: 10px;
  left: 25%;
  z-index: 5;
  background-color: #fff;
  padding: 5px;
  border: 1px solid #999;
  text-align: center;
  font-family: "Roboto", "sans-serif";
  line-height: 30px;
  padding-left: 10px;
}
#right-panel {
  font-family: "Roboto", "sans-serif";
  line-height: 30px;
  padding-left: 10px;
}

.location-buttons {
  width: 100%;
}

#right-panel select,
#right-panel input {
  font-size: 15px;
}

#right-panel select {
  width: 100%;
}

#right-panel i {
  font-size: 12px;
}

.maps {
  margin-right: 20%;
}

#type {

}

#floating-panel {
  background: #fff;
  padding: 5px;
  font-size: 14px;
  font-family: Arial;
  border: 1px solid #ccc;
  box-shadow: 0 2px 2px rgba(33, 33, 33, 0.4);
  display: none;
}

.hover-mouse:hover {
  cursor: pointer;
}

h4 {
  text-align: center;
  font-weight: bold;
}

.center-text {
  text-align: center;
}

.directions-error {
  color: red;
}
@media print {
  #map {
    height: 500px;
    margin: 0;
  }
  #right-panel {
    float: none;
    width: auto;
  }
}
</style>