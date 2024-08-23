<template>
  <div class="background">
   <span></span>
   <span></span>
   <span></span>
   <span></span>
   <span></span>
   <span></span>
   <span></span>
   <span></span>
   <span></span>
   <span></span>
   <span></span>
   <span></span>
   <span></span>
   <span></span>
   <span></span>
   <span></span>
   <span></span>
   <span></span>
   <span></span>
   <span></span>
   <span></span>
   <span></span>
   <span></span>
   <span></span>
   <span></span>
   <span></span>
   <span></span>
   <span></span>
   <span></span>
   </div>
  <div class="login-container" v-if="!isLoggedIn">
    <button class="spotify-login-button" @click="tryLogin" >Login with Spotify</button>
  </div> 
  
  <div v-if="loading" class="loading">
    Loading...
  </div>
  <div v-else> 
    <div class="container" v-if="isLoggedIn">
      <div class="top-artists" v-if="artistView">
      <div class="button-group">
        <button
          v-for="(label, index) in timePeriodLabels"
          :key="index"
          @click="updateTimePeriod(timePeriods[index])"
          :class="{'time-period': true, active: timePeriods[index] === selectedTimePeriod }"
          class="button"
        >
          {{ label }}
        </button>
      </div>
        
      <h2>Top Artists</h2>

      <table>
        <thead>
          <tr>
            <th>Popularity</th>
            <th>Name</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="artist in topArtists" :key="artist.id">
            <td>{{ artist.popularity }}</td>
            <td>{{ artist.name }}</td>
          </tr>
        </tbody>
      </table>

      <h3>Average popularity: {{ avgPopularity }}</h3>
          
      <button class="nav-button" @click="fetchRelatedArtistsForArtists(); artistView=false">Fetch rec artists</button>
     </div>
      <div v-else class="recs">
        <h2>Recommendations</h2>
        <table>
          <thead>
            <tr>
              <th>Artist</th>
              <th>Popularity</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="rec in recArtists" :key="rec.id">
              <td>{{ rec.name }}</td>
              <td>{{ rec.popularity }}</td>
            </tr>
          </tbody>
        </table>
        <button class="nav-button" @click="artistView=true">Back to artists</button>
      </div>
    </div>
</div>
</template>

<script setup>
import { ref, onMounted, computed, watch } from 'vue';
import Chart from 'chart.js/auto';

const topItems = ref([]);
const userGreeting = ref('');
const isLoggedIn = ref(false);
const timePeriods = ['long_term', 'medium_term', 'short_term'];
const timePeriodLabels = ['Past Year', 'Past 6 Months', 'Past Month'];
const selectedTimePeriod = ref('short_term');
const topType = ref('artists');
const recommendations = ref([]);
const selectedNumTracks = ref(50);
const recommendedTracks = ref([]);
const artistData = ref({});
const loading = ref(true);
const recArtists = ref({});
const artistView = ref(true);


const seedArtists = computed(() => {
  if (topType.value === 'artists') {
    return topArtists.value.map(item => item.id).join(',');
  }
  return '';
});

const avgPopularity = computed(() => {
  const items = topArtists.value;
  const totalPopularity = items.reduce((sum, item) => sum + item.popularity, 0);
  return (items.length > 0 ? totalPopularity / items.length : 0).toFixed(1);
});

const topArtists = computed(() => {
  return artistData.value.slice(0, 10);
}); 
    
const nicheArtists = computed(() => {
  let items = [...artistData.value].sort((a, b) => a.popularity - b.popularity);
  return items.slice(0, 5);
});

const updateTimePeriod = (period) => {
  selectedTimePeriod.value = period;
};
      
function tryLogin() {
  const clientId = '650edf944c264da5aa11c5f94b19db12';
  const redirectUri = process.env.NODE_ENV === 'production'
    ? 'https://aretter329.github.io/niche-artist-finder/'
    : 'http://localhost:5173/';
  const scopes = 'user-read-private user-read-email user-top-read';
  const url = `https://accounts.spotify.com/authorize?response_type=token&client_id=${clientId}&redirect_uri=${encodeURIComponent(redirectUri)}&scope=${encodeURIComponent(scopes)}`;
  window.location.href = url;
}

function getHashParams() {
  const hash = window.location.hash.substring(1);
  return hash.split('&').reduce((result, item) => {
    const parts = item.split('=');
    result[parts[0]] = decodeURIComponent(parts[1]);
    return result;
  }, {});
}

function buildUrl(base, params) {
  const query = new URLSearchParams(params).toString();
  return `${base}?${query}`;
}

async function fetchUserData(accessToken) {
  try {
    const response = await fetch('https://api.spotify.com/v1/me/', {
      headers: {
        Authorization: `Bearer ${accessToken}`
      }
    });

    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }

    const data = await response.json();
    userGreeting.value = `Hello, ${data.display_name}!`;
    isLoggedIn.value = true;

  } catch (error) {
    console.error('Error fetching user data:', error);
  }
}

async function fetchTopArtists(accessToken) {
  if (accessToken) {
    try {
      const params = {
          time_range: selectedTimePeriod.value,
          limit: selectedNumTracks.value
        };
      const baseUrl = `https://api.spotify.com/v1/me/top/artists`;
      const url = buildUrl(baseUrl, params);
      const response = await fetch(url, {
          headers: {
            Authorization: `Bearer ${accessToken}`
          }
        });

      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }

      const data = await response.json();

      artistData.value = data.items.map(item => ({
        name: item.name,
        popularity: item.popularity,
        id: item.id
      }));
    }  

     catch (error) {
      console.error('Error fetching top tracks:', error);
    }
  } else {
    console.log('No access token provided');
  }
}

async function fetchRelatedArtistsForArtists() {
  const hashParams = getHashParams();
  const accessToken = hashParams.access_token;

  if (!accessToken) {
    console.error('Access token is missing.');
    return;
  }

  try {
    let allArtists = topArtists.value;
    let leastPopularArtists = [];
    const popularityThreshold = 80; // Popularity threshold for filtering artists
    const batchSize = 50;
    let offset = 0;

    // Start with the least popular artists from the initial list
    leastPopularArtists = allArtists
      .sort((a, b) => a.popularity - b.popularity)
      .slice(0, 10);


    console.log('HERE');
    let finalArtists = [];

    while (finalArtists.length < 10) {
      // Fetch related artists for the current least popular artists
      const relatedArtistsPromises = leastPopularArtists.map(async (artist) => {
        const relatedResponse = await fetch(`https://api.spotify.com/v1/artists/${artist.id}/related-artists`, {
          headers: {
            Authorization: `Bearer ${accessToken}`
          }
        });

        if (!relatedResponse.ok) {
          throw new Error(`HTTP error! status: ${relatedResponse.status}`);
        }

        const relatedData = await relatedResponse.json();
        return relatedData.artists.map(relatedArtist => ({
          name: relatedArtist.name,
          id: relatedArtist.id,
          popularity: relatedArtist.popularity
        }));
      });

      const relatedArtistsResults = await Promise.all(relatedArtistsPromises);
      let relatedArtists = relatedArtistsResults.flat();

      // Add related artists to the allArtists list and re-sort
      allArtists = [...allArtists, ...relatedArtists];
      allArtists.sort((a, b) => a.popularity - b.popularity);

      // Update leastPopularArtists from allArtists
      leastPopularArtists = allArtists
        .sort((a, b) => a.popularity - b.popularity)
        .slice(0, 10);

      // Update finalArtists with those meeting the popularity criteria
      finalArtists = leastPopularArtists
        .filter(artist => artist.popularity < popularityThreshold)
        .sort((a, b) => a.popularity - b.popularity)
        .slice(0, 10);

      // If no new related artists or if unable to populate the list further, break the loop
      if (relatedArtists.length === 0) {
        break;
      }
    }

    // If still not enough artists, fallback to the initial least popular artists within the threshold
    if (finalArtists.length < 10) {
      finalArtists = leastPopularArtists
        .filter(artist => artist.popularity < popularityThreshold);
    }

    // Set recommendations with the final list of artists
    recArtists.value = finalArtists;

  } catch (error) {
    console.error('Error fetching related artists:', error);
  }
}

/* async function fetchRecommendations() {
  const hashParams = getHashParams();
  const accessToken = hashParams.access_token;

  if (accessToken) {
    try {
      const maxPopularity = 30; 
      const response = await fetch(`https://api.spotify.com/v1/recommendations?seed_artists=${seedArtists.value}&seed_tracks=${seedTracks.value}&max_popularity=${maxPopularity}`, {
        headers: {
          Authorization: `Bearer ${accessToken}`
        }
      });

      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }

      const data = await response.json();

      const artistDetailsPromises = data.tracks.map(async track => {
        const artistId = track.artists[0].id; // Assuming there's only one artist per track
        const artistResponse = await fetch(`https://api.spotify.com/v1/artists/${artistId}`, {
          headers: {
            Authorization: `Bearer ${accessToken}`
          }
        });
        
        if (!artistResponse.ok) {
          throw new Error(`HTTP error fetching artist details! status: ${artistResponse.status}`);
        }

        const artistData = await artistResponse.json();
        return {
          name: track.name,
          artist: track.artists[0].name,
          album: track.album.name,
          id: track.id,
          popularity: track.popularity,
          artistPopularity: artistData.popularity // Include artist's popularity
        };
      });

      recommendations.value = await Promise.all(artistDetailsPromises);

    } catch (error) {
      console.error('Error fetching recommendations:', error);
    }
  }
}
*/
watch(selectedTimePeriod, async (newTimePeriod) => {
  const hashParams = getHashParams();
  const accessToken = hashParams.access_token;

  if (accessToken) {
    try {
      await fetchUserData(accessToken);
      await fetchTopArtists(accessToken);
    } catch (error) {
      console.error('Error during data fetching:', error);
    } finally {
      loading.value = false; // Ensure loading is set to false after data fetching
    }
  } else {
    loading.value = false; // Set loading to false if no access token
  }
});

onMounted(async () => {
  const hashParams = getHashParams();
  const accessToken = hashParams.access_token;

  if (accessToken) {
    try {
      await fetchUserData(accessToken);
      await fetchTopArtists(accessToken);
    } catch (error) {
      console.error('Error during data fetching:', error);
    } finally {
      loading.value = false; // Ensure loading is set to false after data fetching
    }
  } else {
    loading.value = false; // Set loading to false if no access token
  }
  
});


</script>

<style scoped>


.login-container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  background: var(--black); 
  margin: 0;
  position: fixed;
  z-index: 2;
  top: 50%; /* Position it vertically in the middle */
  left: 50%; /* Position it horizontally in the middle */
  transform: translate(-50%, -50%); 
}

.spotify-login-button {
  background: var(--spotify-green); 
  border: none;
  color: white;
  padding: 15px 30px;
  font-size: 16px;
  font-weight: bold;
  text-transform: uppercase;
  border-radius: 30px;
  cursor: pointer;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.3);
}

.spotify-login-button:hover {
  background: var(--spotify-green-lighter); 
  transform: scale(1.05); 
}

/* Focus effect for the button */
.spotify-login-button:focus {
  outline: none; /* Remove default focus outline */
}

/* Active effect for the button */
.spotify-login-button:active {
  background: var(--spotify-green-dark); /* Even darker green on click */
  transform: scale(0.98); /* Slight scale down */
}

th, td{
  text-align: center;
}
.button-group {
  display: flex;
  gap: 8px;
  justify-content: center;
}

.time-period {
  background: rgba(255, 118, 244, 0.7);
  color: black;
  padding: 5px 10px;
  font-size: 16px;
  cursor: pointer;
  border-radius: 4px;
  transition: color 0.3s, border-color 0.3s;
}

.time-period:hover {
  background-color: #f987ff;
  color: white;
}

.time-period.active {
  background-color: #dc60c1;
  color: white;
}

@keyframes move {
    100% {
        transform: translate3d(0, 0, 1px) rotate(360deg);
    }
}

.background {
    position: fixed;
    width: 100vw;
    height: 100vh;
    top: 0;
    left: 0;
    background: #000000;
    overflow: hidden;
    z-index: 1;
}

.background span {
    width: 50vmin;
    height: 50vmin;
    border-radius: 50vmin;
    backface-visibility: hidden;
    position: absolute;
    animation: move;
    animation-duration: 20;
    animation-timing-function: linear;
    animation-iteration-count: infinite;
}
.loading, .container {
  font-family: 'Gill Sans', 'Gill Sans MT', Calibri, 'Trebuchet MS', sans-serif ;
  position: fixed; /* Position it fixed to keep it on the screen */
  top: 50%; /* Position it vertically in the middle */
  left: 50%; /* Position it horizontally in the middle */
  transform: translate(-50%, -50%); /* Offset by half its width and height to truly center it */
  z-index: 2; /* Ensure it is on top of other content */
  background: rgba(0, 0, 0, 0.7); /* Black background with 70% opacity */
  color: white; /* Text color */
  width: 70%; /* Width of the container */
  padding: 20px; /* Optional: Adds padding inside the container */
  box-sizing: border-box; /* Includes padding and border in the element's total width and height */
  border-radius: 8px; /* Optional: Rounds the corners of the container */
  display: flex; 
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.top-artists, .recs{
  display: flex; 
  align-items: center;
  justify-content: center;
  flex-direction: column;
}

.nav-button {
  background: rgba(118, 230, 255, 0.7);
  color: black;
  padding: 5px 10px;
  font-size: 16px;
  cursor: pointer;
  border-radius: 4px;
  transition: color 0.3s, border-color 0.3s;
}

.nav-button:hover {
  background-color: #3095aa;
  color: white;
}

.background span:nth-child(0) {
    color: #f43ea8;
    top: 11%;
    left: 81%;
    animation-duration: 10s;
    animation-delay: -53s;
    transform-origin: -9vw 15vh;
    box-shadow: 100vmin 0 13.104205727615401vmin currentColor;
}
.background span:nth-child(1) {
    color: #3c7d86;
    top: 47%;
    left: 100%;
    animation-duration: 41s;
    animation-delay: -257s;
    transform-origin: -21vw 19vh;
    box-shadow: 100vmin 0 13.4304545082933vmin currentColor;
}
.background span:nth-child(2) {
    color: #f43ea8;
    top: 14%;
    left: 4%;
    animation-duration: 162s;
    animation-delay: -32s;
    transform-origin: 16vw 5vh;
    box-shadow: -100vmin 0 12.651743457048411vmin currentColor;
}
.background span:nth-child(3) {
    color: #93fba8;
    top: 94%;
    left: 8%;
    animation-duration: 109s;
    animation-delay: -236s;
    transform-origin: 22vw 23vh;
    box-shadow: 100vmin 0 13.040588449573665vmin currentColor;
}
.background span:nth-child(4) {
    color: #f43ea8;
    top: 71%;
    left: 41%;
    animation-duration: 210s;
    animation-delay: -282s;
    transform-origin: -16vw 2vh;
    box-shadow: -100vmin 0 13.044825807546486vmin currentColor;
}
.background span:nth-child(5) {
    color: #f43ea8;
    top: 48%;
    left: 32%;
    animation-duration: 93s;
    animation-delay: -236s;
    transform-origin: -18vw -4vh;
    box-shadow: -100vmin 0 13.21108266767685vmin currentColor;
}
.background span:nth-child(6) {
    color: #f43ea8;
    top: 79%;
    left: 42%;
    animation-duration: 250s;
    animation-delay: -46s;
    transform-origin: 11vw 14vh;
    box-shadow: 100vmin 0 12.646815815828045vmin currentColor;
}
.background span:nth-child(7) {
    color: #f43ea8;
    top: 46%;
    left: 78%;
    animation-duration: 303s;
    animation-delay: -212s;
    transform-origin: -8vw -4vh;
    box-shadow: 100vmin 0 13.032543508159788vmin currentColor;
}
.background span:nth-child(8) {
    color: #f43ea8;
    top: 44%;
    left: 33%;
    animation-duration: 33s;
    animation-delay: -19s;
    transform-origin: 9vw 4vh;
    box-shadow: 100vmin 0 13.05224857184325vmin currentColor;
}
.background span:nth-child(9) {
    color: #f43ea8;
    top: 75%;
    left: 90%;
    animation-duration: 255s;
    animation-delay: -265s;
    transform-origin: -23vw 19vh;
    box-shadow: -100vmin 0 12.703725449086495vmin currentColor;
}
.background span:nth-child(10) {
    color: #f43ea8;
    top: 31%;
    left: 7%;
    animation-duration: 81s;
    animation-delay: -42s;
    transform-origin: -18vw -21vh;
    box-shadow: -100vmin 0 12.83303729987308vmin currentColor;
}
.background span:nth-child(11) {
    color: #f43ea8;
    top: 48%;
    left: 91%;
    animation-duration: 184s;
    animation-delay: -2s;
    transform-origin: -17vw -11vh;
    box-shadow: -100vmin 0 12.559744643163318vmin currentColor;
}
.background span:nth-child(12) {
    color: #3c7d86;
    top: 20%;
    left: 99%;
    animation-duration: 51s;
    animation-delay: -261s;
    transform-origin: 3vw -16vh;
    box-shadow: -100vmin 0 12.575619185992469vmin currentColor;
}
.background span:nth-child(13) {
    color: #f43ea8;
    top: 71%;
    left: 23%;
    animation-duration: 137s;
    animation-delay: -199s;
    transform-origin: 21vw -17vh;
    box-shadow: 100vmin 0 12.830449295893747vmin currentColor;
}
.background span:nth-child(14) {
    color: #f43ea8;
    top: 38%;
    left: 8%;
    animation-duration: 90s;
    animation-delay: -168s;
    transform-origin: 6vw 14vh;
    box-shadow: -100vmin 0 12.899840145472066vmin currentColor;
}
.background span:nth-child(15) {
    color: #93fba8;
    top: 10%;
    left: 43%;
    animation-duration: 105s;
    animation-delay: -239s;
    transform-origin: -1vw 8vh;
    box-shadow: -100vmin 0 12.67179305304669vmin currentColor;
}
.background span:nth-child(16) {
    color: #93fba8;
    top: 99%;
    left: 25%;
    animation-duration: 222s;
    animation-delay: -265s;
    transform-origin: 24vw -13vh;
    box-shadow: 100vmin 0 13.312778486739262vmin currentColor;
}
.background span:nth-child(17) {
    color: #3c7d86;
    top: 39%;
    left: 91%;
    animation-duration: 111s;
    animation-delay: -69s;
    transform-origin: -10vw -10vh;
    box-shadow: -100vmin 0 13.39815305225424vmin currentColor;
}
.background span:nth-child(18) {
    color: #3c7d86;
    top: 97%;
    left: 98%;
    animation-duration: 65s;
    animation-delay: -93s;
    transform-origin: -2vw 5vh;
    box-shadow: 100vmin 0 13.301439543581559vmin currentColor;
}
.background span:nth-child(19) {
    color: #f43ea8;
    top: 24%;
    left: 69%;
    animation-duration: 48s;
    animation-delay: -278s;
    transform-origin: 13vw -5vh;
    box-shadow: -100vmin 0 13.075683628849106vmin currentColor;
}
.background span:nth-child(20) {
    color: #93fba8;
    top: 15%;
    left: 14%;
    animation-duration: 219s;
    animation-delay: -21s;
    transform-origin: 23vw 12vh;
    box-shadow: 100vmin 0 13.337491705630411vmin currentColor;
}
.background span:nth-child(21) {
    color: #f43ea8;
    top: 6%;
    left: 55%;
    animation-duration: 92s;
    animation-delay: -290s;
    transform-origin: 22vw -20vh;
    box-shadow: 100vmin 0 13.15413932417116vmin currentColor;
}
.background span:nth-child(22) {
    color: #f43ea8;
    top: 8%;
    left: 57%;
    animation-duration: 169s;
    animation-delay: -29s;
    transform-origin: 1vw 24vh;
    box-shadow: -100vmin 0 13.341916399010882vmin currentColor;
}
.background span:nth-child(23) {
    color: #93fba8;
    top: 95%;
    left: 10%;
    animation-duration: 203s;
    animation-delay: -248s;
    transform-origin: 6vw -11vh;
    box-shadow: 100vmin 0 12.808795282426392vmin currentColor;
}
.background span:nth-child(24) {
    color: #f43ea8;
    top: 21%;
    left: 32%;
    animation-duration: 22s;
    animation-delay: -43s;
    transform-origin: -3vw -5vh;
    box-shadow: 100vmin 0 13.423185335830462vmin currentColor;
}
.background span:nth-child(25) {
    color: #93fba8;
    top: 55%;
    left: 92%;
    animation-duration: 221s;
    animation-delay: -228s;
    transform-origin: 17vw 11vh;
    box-shadow: 100vmin 0 12.742907806250141vmin currentColor;
}
.background span:nth-child(26) {
    color: #93fba8;
    top: 73%;
    left: 14%;
    animation-duration: 30s;
    animation-delay: -223s;
    transform-origin: 1vw -24vh;
    box-shadow: 100vmin 0 12.782533263341094vmin currentColor;
}
.background span:nth-child(27) {
    color: #3c7d86;
    top: 74%;
    left: 64%;
    animation-duration: 18s;
    animation-delay: -186s;
    transform-origin: 16vw 22vh;
    box-shadow: 100vmin 0 13.03601475648522vmin currentColor;
}
.background span:nth-child(28) {
    color: #93fba8;
    top: 92%;
    left: 73%;
    animation-duration: 164s;
    animation-delay: -45s;
    transform-origin: 21vw 6vh;
    box-shadow: 100vmin 0 12.623149084425318vmin currentColor;
}


</style>
