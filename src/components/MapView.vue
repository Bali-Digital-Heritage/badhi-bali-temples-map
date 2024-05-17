<template>
  <div class="corner info p-4">
    <h2 class="title">Bali Temples Map</h2>
    <p>
      The data is from
      <a
        href="https://github.com/KadekSatriadi/OSM-Bali-temples"
        target="_blank"
        >Bali Temples </a
      >.
    </p>
    <p>
      <small
        ><i
          >Developed by Kadek Satriadi, under the
          <a href="https://badhi.id" target="_blank">BADHI project</a>.</i
        ></small
      >
    </p>
    <div class="legend mt-3">
      <div class="legend-item">
        <div class="legend-icon named"></div><div class="legend-label">Named point</div>
      </div> 
      <div class="legend-item">
        <div class="legend-icon unnamed"></div><div class="legend-label">Unnamed point</div>
      </div> 

    </div>
    <hr />
    <button @click="plotData" class="button  mb-4">Fetch and Plot</button>
    <div class="buttons">
      <button class="button is-small" @click="changeTileLayer('Standard')">Standard</button>
      <button class="button is-small" @click="changeTileLayer('Hot')">Hot</button>
      <button class="button is-small" @click="changeTileLayer('OpenTopoMap')">OpenTopoMap</button>
      <button class="button is-small" @click="changeTileLayer('WorldImagery')">WorldImagery</button>
      <!-- Add more buttons for other styles if needed -->
    </div>
  </div>
  <div id="map" ></div>
</template>

<style>
.popup {
  max-height: 200px;
  overflow-y: scroll;
}
.legend{
  display: flex;
}
.legend-item{
  display: flex;
  margin-right: 5px;
  padding: 2px;
}
.unnamed{
  background-color: #7570b3;
}
.named{
  background-color: #d95f02;
}
.legend-icon{
  width: 24px;
  height: 24px;
  border: 1px solid black;
  display: block;
  border-radius: 50%;
}
.legend-label{
  display: block;
  padding-left: 5px;
}
#map {
  width: 100%;
  height: 100vh !important;
}
.corner {
  position: absolute !important;
  top: 5%;
  z-index: 1000;
  right: 1%;
}
.info {
  background-color: rgba(255, 255, 255, 0.6) !important;
  border-radius: 5px;
}
</style>

<script lang="ts">
import { onMounted, ref } from "vue";
import L from "leaflet";
import axios from "axios";
import Papa from "papaparse";
import "leaflet/dist/leaflet.css";

export default {
  name: "MapView",
  setup() {
    const localStorageKey = "geospatialData";

    const map = ref(null);
    const csvUrl =
      "https://raw.githubusercontent.com/KadekSatriadi/OSM-Bali-temples/main/Bali_place_of_worship_all.csv";


    onMounted(async () => {
      // Initialize the map
      map.value = L.map("map").setView([-8.369663, 115.190955], 10);

      // Set up the tile layer
      L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
        attribution: "&copy; OpenStreetMap contributors",
      }).addTo(map.value);

      let csvDataString = localStorage.getItem(localStorageKey);
      //not available, fetch
      if (csvDataString) {
        plotData();
      }
    });

    const fetchData = async () => {
      // Fetch the CSV data
      let csvData;
      console.log("fecthing data ", csvUrl);
      try {
        const response = await axios.get(csvUrl);
        csvData = response.data;
        localStorage.setItem(localStorageKey, csvData);
        console.log("Success");
      } catch (error) {
        console.error("Error fetching the CSV data", error);
      }

      return csvData;
    };
    const createPopupContent = (row: any) => {
      let table = "<div class='popup'><table>";
      for (const key in row) {
        if (row.hasOwnProperty(key)) {
          table += `<tr><th>${key}</th><td>${row[key]}</td></tr>`;
        }
      }
      table += "</table></div>";
      return table;
    };

    const plotString = (csvDataString: any) => {
      // Parse the CSV data
      Papa.parse(csvDataString, {
        header: true,
        complete: (results: any) => {
          results.data.forEach((row: any) => {
            const lat = parseFloat(row.Y);
            const lon = parseFloat(row.X);
            let color;
            // Check if the value is NA
            if (row.name === "NA") {
              // Color for NA values
              color = "#7570b3";
            } else {
              // Color for non-NA values
              color = "#d95f02";
            }
            // console.log("lat", lat, "long", lon);
            if (!isNaN(lat) && !isNaN(lon)) {
              //L.marker([lat, lon]).addTo(map.value);
              L.circle([lat, lon], {
                color: color, // Circle border color
                fillColor: color, // Circle fill color
                fillOpacity: 0.5, // Circle fill opacity
                radius: 200, // Circle radius in meters
              })
                .addTo(map.value)
                .bindPopup(createPopupContent(row));
            }
          });
        },
      });
    };

    const plotData = () => {
      //check local
      let csvDataString = localStorage.getItem(localStorageKey);
      //not available, fetch
      if (!csvDataString) {
        console.log("data not found in local storage, fetch");
        (async () => {
          // Your async code here
          fetchData().then((data) => {
            plotString(data);
          });
        })();
      } else {
        plotString(csvDataString);
        console.log("load data from local storage, fetch");
      }
    };

    const changeTileLayer = (layerName:any) => {

      // Set up the new tile layer based on the selected style
      switch (layerName) {
        case 'Standard':
          L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
            attribution: '&copy; OpenStreetMap contributors'
          }).addTo(map.value);
          break;
        case 'Hot':
          L.tileLayer('https://{s}.tile.openstreetmap.fr/hot/{z}/{x}/{y}.png', {
            attribution: '&copy; OpenStreetMap contributors, Tiles style by <a href="https://www.hotosm.org/">Humanitarian OpenStreetMap Team</a>'
          }).addTo(map.value);
          break;
        case 'OpenTopoMap':
          L.tileLayer('https://{s}.tile.opentopomap.org/{z}/{x}/{y}.png', {
            attribution: '&copy; OpenTopoMap'
          }).addTo(map.value);
          break;
        case 'WorldImagery':
          L.tileLayer('https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}', {
            attribution: '&copy; Esri, DigitalGlobe, GeoEye, Earthstar Geographics, CNES/Airbus DS, USDA, USGS, AeroGRID, IGN, and the GIS User Community'
          }).addTo(map.value);
          break;
        // Add more cases for other styles if needed
        default:
          break;
      }
    };

    return { fetchData, plotData, changeTileLayer };
  },
};
</script>

<style scoped>
#map {
  width: 100%;
  height: 500px;
}
</style>
