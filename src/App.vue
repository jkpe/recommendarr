<template>
  <div class="app-container">
    <header class="app-header">
      <img alt="App logo" src="./assets/logo.png" class="logo">
      <h1>Recommendarr</h1>
    </header>
    
    <main>
      <div v-if="!sonarrConnected && !radarrConnected">
        <p class="choose-service">Choose a service to connect to:</p>
        <div class="service-buttons">
          <button class="service-button" @click="showSonarrConnect = true">
            Connect to Sonarr
            <small>For TV recommendations</small>
          </button>
          <button class="service-button" @click="showRadarrConnect = true">
            Connect to Radarr
            <small>For movie recommendations</small>
          </button>
        </div>
      </div>
      
      <SonarrConnection v-if="showSonarrConnect && !sonarrConnected" @connected="handleSonarrConnected" />
      <RadarrConnection v-if="showRadarrConnect && !radarrConnected" @connected="handleRadarrConnected" />
      
      <div v-if="sonarrConnected || radarrConnected">
        <AppNavigation 
          :activeTab="activeTab" 
          @navigate="handleNavigate"
          @logout="handleLogout" 
        />
        
        <div class="content">
          <TVRecommendations 
            v-if="activeTab === 'tv-recommendations'" 
            :series="series"
            :sonarrConfigured="sonarrConnected"
            @navigate="handleNavigate" 
          />
          
          <MovieRecommendations 
            v-if="activeTab === 'movie-recommendations'" 
            :movies="movies"
            :radarrConfigured="radarrConnected"
            @navigate="handleNavigate" 
          />
          
          <AISettings
            v-if="activeTab === 'settings'"
            @settings-updated="handleSettingsUpdated"
            @sonarr-settings-updated="handleSonarrSettingsUpdated"
            @radarr-settings-updated="handleRadarrSettingsUpdated"
          />
        </div>
      </div>
    </main>
  </div>
</template>

<script>
import SonarrConnection from './components/SonarrConnection.vue'
import RadarrConnection from './components/RadarrConnection.vue'
import AppNavigation from './components/Navigation.vue'
import TVRecommendations from './components/TVRecommendations.vue'
import MovieRecommendations from './components/MovieRecommendations.vue'
import AISettings from './components/AISettings.vue'
import sonarrService from './services/SonarrService'
import radarrService from './services/RadarrService'

export default {
  name: 'App',
  components: {
    SonarrConnection,
    RadarrConnection,
    AppNavigation,
    TVRecommendations,
    MovieRecommendations,
    AISettings
  },
  data() {
    return {
      sonarrConnected: false,
      radarrConnected: false,
      showSonarrConnect: false,
      showRadarrConnect: false,
      activeTab: 'tv-recommendations',
      series: [],
      movies: []
    }
  },
  created() {
    // Check if Sonarr is already configured on startup
    if (sonarrService.isConfigured()) {
      this.checkSonarrConnection();
    }
    
    // Check if Radarr is already configured on startup
    if (radarrService.isConfigured()) {
      this.checkRadarrConnection();
    }
  },
  methods: {
    async checkSonarrConnection() {
      try {
        const success = await sonarrService.testConnection();
        if (success) {
          this.sonarrConnected = true;
          this.fetchSeriesData();
          if (this.activeTab === 'movie-recommendations' && !this.radarrConnected) {
            this.activeTab = 'tv-recommendations';
          }
        }
      } catch (error) {
        console.error('Failed to connect with stored Sonarr credentials:', error);
      }
    },
    
    async checkRadarrConnection() {
      try {
        const success = await radarrService.testConnection();
        if (success) {
          this.radarrConnected = true;
          this.fetchMoviesData();
          if (this.activeTab === 'tv-recommendations' && !this.sonarrConnected) {
            this.activeTab = 'movie-recommendations';
          }
        }
      } catch (error) {
        console.error('Failed to connect with stored Radarr credentials:', error);
      }
    },
    handleSonarrConnected() {
      this.sonarrConnected = true;
      this.showSonarrConnect = false;
      this.fetchSeriesData();
      this.activeTab = 'tv-recommendations';
    },
    
    handleRadarrConnected() {
      this.radarrConnected = true;
      this.showRadarrConnect = false;
      this.fetchMoviesData();
      this.activeTab = 'movie-recommendations';
    },
    handleNavigate(tab) {
      this.activeTab = tab;
      
      // If we're switching to TV recommendations, ensure we have series data
      if (tab === 'tv-recommendations' && this.series.length === 0 && this.sonarrConnected) {
        this.fetchSeriesData();
      }
      
      // If we're switching to movie recommendations, ensure we have movies data
      if (tab === 'movie-recommendations' && this.movies.length === 0 && this.radarrConnected) {
        this.fetchMoviesData();
      }
    },
    async fetchSeriesData() {
      try {
        this.series = await sonarrService.getSeries();
      } catch (error) {
        console.error('Failed to fetch series data for recommendations:', error);
      }
    },
    
    async fetchMoviesData() {
      try {
        this.movies = await radarrService.getMovies();
      } catch (error) {
        console.error('Failed to fetch movies data for recommendations:', error);
      }
    },
    handleSettingsUpdated() {
      // When AI settings are updated, show a brief notification or just stay on the settings page
      console.log('AI settings updated successfully');
    },
    
    handleSonarrSettingsUpdated() {
      // Check the Sonarr connection with the new settings
      this.checkSonarrConnection();
      console.log('Sonarr settings updated, testing connection');
    },
    
    handleRadarrSettingsUpdated() {
      // Check the Radarr connection with the new settings
      this.checkRadarrConnection();
      console.log('Radarr settings updated, testing connection');
    },
    handleLogout() {
      // Clear all stored credentials
      localStorage.removeItem('sonarrBaseUrl');
      localStorage.removeItem('sonarrApiKey');
      localStorage.removeItem('radarrBaseUrl');
      localStorage.removeItem('radarrApiKey');
      localStorage.removeItem('openaiApiKey');
      localStorage.removeItem('openaiModel');
      
      // Reset service configurations
      sonarrService.configure('', '');
      radarrService.configure('', '');
      
      // Reset UI state
      this.sonarrConnected = false;
      this.radarrConnected = false;
      this.series = [];
      this.movies = [];
      this.showSonarrConnect = false;
      this.showRadarrConnect = false;
      this.activeTab = 'tv-recommendations';
    }
  }
}
</script>

<style>
:root {
  /* Light theme (default) */
  --bg-color: #f5f5f5;
  --main-bg-color: #ffffff;
  --text-color: #2c3e50;
  --header-color: #2c3e50;
  --border-color: #ddd;
  --card-bg-color: #ffffff;
  --card-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  --nav-bg-color: #2c3e50;
  --nav-text-color: #ccc;
  --nav-active-bg: rgba(255, 255, 255, 0.2);
  --nav-hover-bg: rgba(255, 255, 255, 0.1);
  --nav-active-text: #ffffff;
  --input-bg: #ffffff;
  --input-border: #ddd;
  --input-text: #333;
  --button-primary-bg: #4CAF50;
  --button-primary-text: white;
  --button-secondary-bg: #f0f0f0;
  --button-secondary-text: #333;
  
  /* Transition for theme changes */
  --transition-speed: 0.3s;
}

body.dark-theme {
  /* Dark theme */
  --bg-color: #1a1a1a;
  --main-bg-color: #2a2a2a;
  --text-color: #e0e0e0;
  --header-color: #e0e0e0;
  --border-color: #444;
  --card-bg-color: #333;
  --card-shadow: 0 2px 10px rgba(0, 0, 0, 0.3);
  --nav-bg-color: #222;
  --nav-text-color: #aaa;
  --nav-active-bg: rgba(255, 255, 255, 0.15);
  --nav-hover-bg: rgba(255, 255, 255, 0.05);
  --nav-active-text: #ffffff;
  --input-bg: #3a3a3a;
  --input-border: #555;
  --input-text: #e0e0e0;
  --button-primary-bg: #388E3C;
  --button-primary-text: white;
  --button-secondary-bg: #444;
  --button-secondary-text: #e0e0e0;
}

body {
  margin: 0;
  background-color: var(--bg-color);
  color: var(--text-color);
  transition: background-color var(--transition-speed), color var(--transition-speed);
}

#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: var(--text-color);
}

.app-container {
  max-width: 1200px;
  width: 100%;
  margin: 0 auto;
  padding: 10px;
  box-sizing: border-box;
}

@media (min-width: 480px) {
  .app-container {
    padding: 20px;
  }
}

.app-header {
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 20px;
  flex-wrap: wrap;
  padding: 20px 0 0 0;
}

@media (min-width: 480px) {
  .app-header {
    margin-bottom: 30px;
  }
}

.logo {
  height: 40px;
  margin-right: 10px;
  transition: filter var(--transition-speed);
}

@media (min-width: 480px) {
  .logo {
    height: 60px;
    margin-right: 15px;
  }
}

body.dark-theme .logo {
  filter: brightness(0.9);
}

h1 {
  margin: 0;
  font-size: 22px;
  color: var(--header-color);
  transition: color var(--transition-speed);
  font-weight: 600;
}

@media (min-width: 480px) {
  h1 {
    font-size: 28px;
  }
}

main {
  background-color: var(--main-bg-color);
  border-radius: 8px;
  box-shadow: var(--card-shadow);
  overflow: hidden;
  transition: background-color var(--transition-speed), box-shadow var(--transition-speed);
}

.content {
  min-height: 400px;
}

.choose-service {
  text-align: center;
  margin: 30px 0 20px;
  font-size: 18px;
  color: var(--text-color);
}

.service-buttons {
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  gap: 20px;
  margin-bottom: 30px;
}

.service-button {
  background-color: var(--card-bg-color);
  border: 2px solid var(--button-primary-bg);
  border-radius: 8px;
  padding: 15px;
  color: var(--text-color);
  font-size: 15px;
  font-weight: bold;
  cursor: pointer;
  transition: all 0.3s ease;
  display: flex;
  flex-direction: column;
  align-items: center;
  min-width: 150px;
  flex: 1;
  max-width: 250px;
}

@media (min-width: 480px) {
  .service-button {
    padding: 20px 30px;
    font-size: 16px;
  }
}

.service-button:hover {
  background-color: var(--button-primary-bg);
  color: var(--button-primary-text);
  transform: translateY(-3px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

.service-button small {
  font-weight: normal;
  opacity: 0.8;
  margin-top: 5px;
  font-size: 12px;
}
</style>
