<template>
  <div class="corner">
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
    <hr />
    <button @click="plotData" class="button is-success">Fetch and Plot</button>
  </div>
  <div id="map"></div>
</template>

<style>
.popup {
  max-height: 200px;
  overflow-y: scroll;
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
          console.log("Parsed data", results.data);
          results.data.forEach((row: any) => {
            const lat = parseFloat(row.Y);
            const lon = parseFloat(row.X);
            // console.log("lat", lat, "long", lon);
            if (!isNaN(lat) && !isNaN(lon)) {
              //L.marker([lat, lon]).addTo(map.value);
              L.circle([lat, lon], {
                color: "red", // Circle border color
                fillColor: "#f03", // Circle fill color
                fillOpacity: 0.5, // Circle fill opacity
                radius: 500, // Circle radius in meters
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

    return { fetchData, plotData };
  },
};
</script>

<style scoped>
#map {
  width: 100%;
  height: 500px;
}
</style>
