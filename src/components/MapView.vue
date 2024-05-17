<template>
  <div class="corner info p-5">
    <h2 class="title">Worship Places in Bali</h2>
    <p>
      The data is from
      <a
        href="https://github.com/KadekSatriadi/OSM-Bali-temples"
        target="_blank"
        >Bali Temples </a
      > (OpenStreetMap).
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

    <div class="legend mt-3">
      <div class="legend-item">
        <div class="legend-icon named"></div>
        <div class="legend-label">Named point ({{ totalnamed }})</div>
      </div>
      <div class="legend-item">
        <div class="legend-icon unnamed"></div>
        <div class="legend-label">Unnamed point ( {{ totalunnamed }})</div>
      </div>
    </div>
    <!-- buggy
      <div class="buttons">
      <button class="button is-small" @click="changeTileLayer('Standard')">Standard</button>
      <button class="button is-small" @click="changeTileLayer('Hot')">Hot</button>
      <button class="button is-small" @click="changeTileLayer('OpenTopoMap')">OpenTopoMap</button>
      <button class="button is-small" @click="changeTileLayer('WorldImagery')">WorldImagery</button>
    </div>
  --></div>
  <div id="map"></div>
</template>

<style>
.popup {
  max-height: 200px;
  overflow-y: scroll;
}
.legend {
  display: flex;
}
.legend-item {
  display: flex;
  margin-right: 5px;
  padding: 2px;
}
.unnamed {
  background-color: #7570b3;
}
.named {
  background-color: #d95f02;
}
.legend-icon {
  width: 24px;
  height: 24px;
  border: 1px solid black;
  display: block;
  border-radius: 50%;
}
.legend-label {
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

<script>
import { onMounted, ref } from "vue";
import L from "leaflet";
import axios from "axios";
import Papa from "papaparse";
import "leaflet/dist/leaflet.css";

export default {
  name: "MapView",
  setup() {
    const localStorageKey = "geospatialData";

    let totalnamed = ref(0);
    let totalunnamed = ref(0);

    const map = ref(null);
    const csvUrl =
      "https://raw.githubusercontent.com/KadekSatriadi/OSM-Bali-temples/main/Bali_place_of_worship_all.csv";

    let tileLayer;

    onMounted(async () => {
      // Initialize the map
      map.value = L.map("map").setView([-8.369663, 115.190955], 10);

      // Set up the tile layer
      tileLayer = L.tileLayer(
        "https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",
        {
          attribution: "&copy; OpenStreetMap contributors",
        }
      ).addTo(map.value);

      plotData();
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
    const createPopupContent = (row) => {
      let table = "<div class='popup'><table>";
      for (const key in row) {
        if (row.hasOwnProperty(key)) {
          table += `<tr><th>${key}</th><td>${row[key]}</td></tr>`;
        }
      }
      table += "</table></div>";
      return table;
    };

    const wktToLatLngs = (wkt) => {
      var coords = wkt.slice(9, -2).split(", ");
      return coords
        .map(function (coord) {
          var latlng = coord.split(" ").map(Number);
          if (isNaN(latlng[0]) || isNaN(latlng[1])) {
            return null;
          }
          return [latlng[1], latlng[0]]; // Leaflet uses [lat, lng]
        })
        .filter(function (latlng) {
          return latlng !== null; // Filter out invalid coordinates
        });
    };

    const plotString = (csvDataString) => {
      // Parse the CSV data
      Papa.parse(csvDataString, {
        header: true,
        complete: (results) => {
          
          results.data.forEach((row) => {
            console.log(row.geometry);
            var latlngs = wktToLatLngs(row.geometry);
            console.log(latlngs);

            let color;
            // Check if the value is NA
            if (row.name === "NA") {
              // Color for NA values
              color = "#7570b3";
              totalunnamed.value += 1;

            } else {
              // Color for non-NA values
              color = "#d95f02";
              totalnamed.value += 1;

            }
            if (latlngs) {
              //L.marker([lat, lon]).addTo(map.value);
              L.polygon(latlngs, {
                color: color, // Circle border color
                fillColor: color, // Circle fill color
                fillOpacity: 0.5, // Circle fill opacity
                radius: 1, // Circle radius in pxl
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

    const changeTileLayer = (layerName) => {
      map.value.removeLayer(tileLayer);

      // Set up the new tile layer based on the selected style
      switch (layerName) {
        case "Standard":
          tileLayer = L.tileLayer(
            "https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",
            {
              attribution: "&copy; OpenStreetMap contributors",
            }
          ).addTo(map.value);
          break;
        case "Hot":
          tileLayer = L.tileLayer(
            "https://{s}.tile.openstreetmap.fr/hot/{z}/{x}/{y}.png",
            {
              attribution:
                '&copy; OpenStreetMap contributors, Tiles style by <a href="https://www.hotosm.org/">Humanitarian OpenStreetMap Team</a>',
            }
          ).addTo(map.value);
          break;
        case "OpenTopoMap":
          tileLayer = L.tileLayer(
            "https://{s}.tile.opentopomap.org/{z}/{x}/{y}.png",
            {
              attribution: "&copy; OpenTopoMap",
            }
          ).addTo(map.value);
          break;
        case "WorldImagery":
          tileLayer = L.tileLayer(
            "https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}",
            {
              attribution:
                "&copy; Esri, DigitalGlobe, GeoEye, Earthstar Geographics, CNES/Airbus DS, USDA, USGS, AeroGRID, IGN, and the GIS User Community",
            }
          ).addTo(map.value);
          break;
        // Add more cases for other styles if needed
        default:
          break;
      }
    };

    return { fetchData, plotData, changeTileLayer, totalnamed, totalunnamed };
  },
};
</script>

<style scoped>
#map {
  width: 100%;
  height: 500px;
}
</style>
