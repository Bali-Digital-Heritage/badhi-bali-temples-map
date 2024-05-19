<template>
  <!-- information panel -->
  <div class="corner info p-5">
    <h2 class="title">Bali Temples</h2>
    <p>
     Worship places and temples in Bali. 
    </p>
    
    <hr />

    <LayerList :layers="layers" @togglelayer="togglelayer" />
    <div class="legend mt-3"></div>
    <!-- buggy
      <div class="buttons">
      <button class="button is-small" @click="changeTileLayer('Standard')">Standard</button>
      <button class="button is-small" @click="changeTileLayer('Hot')">Hot</button>
      <button class="button is-small" @click="changeTileLayer('OpenTopoMap')">OpenTopoMap</button>
      <button class="button is-small" @click="changeTileLayer('WorldImagery')">WorldImagery</button>
    </div>
  -->
  <p>
      <small
        ><i
          >Developed by Kadek Ananta Satriadi, PhD, under the
          <a href="https://badhi.id" target="_blank">BADHI project</a>.<br/>
          Version May 2024.</i
        ></small
      >
    </p>
  
  </div>
  <!-- end of information panel -->
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
import { onMounted, ref, toRaw } from "vue";
import L from "leaflet";
import axios from "axios";
import Papa from "papaparse";
import "leaflet/dist/leaflet.css";
import LayerList from "./LayerList.vue";

export default {
  name: "MapView",
  components: {
    LayerList,
  },
  setup() {
    const map = ref(null);

    const LAYERTYPE = {
      polygon: "polygon",
      point: "point",
    };

    const COLORS = {
      color1: "#7fc97f",
      color2: "#beaed4",
      color3: "#fdc086",
      color4: "#ffff99",
      color5: "#386cb0",
      color6: "#f0027f",
      color7: "#bf5b1",
    };

    const namedColor = COLORS.color2;
    const unnamedColor = COLORS.color1;
    const typeknownColor = COLORS.color5;
    const typeunknownColor = COLORS.color3;

    const templePolygonColorMap = (row, layer) => {
      let color;
      // Check if the value is NA
      if (row.name === "NA") {
        // Color for NA values
        color = namedColor;
      } else {
        // Color for non-NA values
        color = unnamedColor;
      }

      return color;
    };

    const templePointColorMap = (row, layer) => {
      if (row.Type == "") {
        return typeunknownColor;
      } else {
        return typeknownColor;
      }
    };

    const OSMPolygonLayer = {
      type: LAYERTYPE.polygon,
      csvurl:
        "https://raw.githubusercontent.com/KadekSatriadi/OSM-Bali-temples/main/Bali_place_of_worship_all.csv",
      name: "OSM Worship Places",
      description: "Worship places in Bali from Openstreetmap data.",
      geometry: "geometry",
      id: "osmwp",
      isactive: true,
      aes: {
        color: templePolygonColorMap,
      },
      style: {
        fillOpacity: 0.5, // Circle fill opacity
      },
      legend: `<div class="legend-item">
        <div class="legend-icon" style="background-color: ${namedColor}"></div>
        <div class="legend-label">Named place</div>
      </div>
      <div class="legend-item">
        <div class="legend-icon" style="background-color: ${unnamedColor}"></div>
        <div class="legend-label">Unnamed place</div>
      </div>`,
    };

    const TemplesPointLayer = {
      type: LAYERTYPE.point,
      csvurl:
        "https://gist.githubusercontent.com/KadekSatriadi/446b309a444bb7432ca1d3d9f9dac8e0/raw/58196037b0b12b51c1258cf4e473013231cd3f41/Bali_temples_2021.csv",
      name: "Temple Restore data",
      description: "Bali temples data from the Temple Restore project.",
      lat: "Latitude",
      lon: "Longitude",
      id: "ts2021",
      isactive: true,
      aes: {
        color: templePointColorMap,
      },
      style: {
        fillOpacity: 0.5, // Circle fill opacity
        radius: 15, // Circle radius in pxl
      },
      legend: `<div class="legend-item">
        <div class="legend-icon" style="background-color: ${typeknownColor}"></div>
        <div class="legend-label">Type known</div>
      </div>
      <div class="legend-item">
        <div class="legend-icon" style="background-color: ${typeunknownColor}"></div>
        <div class="legend-label">Type unknown</div>
      </div>`,
    };

    const layers = ref([OSMPolygonLayer, TemplesPointLayer]);

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

      layers.value.forEach((layer) => {
        plotData(layer);
      });
      //plotData(layers[1]);
    });

    const fetchData = async (layer) => {
      // Fetch the CSV data
      let csvData;
      try {
        const response = await axios.get(layer.csvurl);
        csvData = response.data;
        localStorage.setItem(layer.id, csvData);
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

    const plotPolygonLayerItem = (row, layer) => {
      var latlngs = wktToLatLngs(row[layer.geometry]);
      let style = layer.style;
      if (layer.aes.color) {
        style.color = layer.aes.color(row, layer);
      }

      if (latlngs) {
        //L.marker([lat, lon]).addTo(map.value);
        const l = L.polygon(latlngs, style)
          .addTo(map.value)
          .bindPopup(createPopupContent(row));

        layer.leaflet.push(l);
      }
    };

    const plotPointLayerItem = (row, layer) => {
      let lat = parseFloat(row[layer.lat]);
      let lon = parseFloat(row[layer.lon]);

      let style = layer.style;
      if (layer.aes.color) {
        style.color = layer.aes.color(row, layer);
      }
      if (lat && lon) {
        //L.marker([lat, lon]).addTo(map.value);
        const l = L.circle([lat, lon], style)
          .addTo(map.value)
          .bindPopup(createPopupContent(row));
        layer.leaflet.push(l);
      }
    };

    const plotString = (csvDataString, layer) => {
      // Parse the CSV data
      layer.leaflet = [];

      Papa.parse(csvDataString, {
        header: true,
        complete: (results) => {
          results.data.forEach((row) => {
            switch (layer.type) {
              case LAYERTYPE.polygon:
                //if data
                if (row[layer.geometry]) {
                  plotPolygonLayerItem(row, layer);
                }
                break;
              case LAYERTYPE.point:
                if (row[layer.lat] && row[layer.lon]) {
                  plotPointLayerItem(row, layer);
                }
                break;
            }
          });
        },
      });
    };

    const plotData = (layer) => {
      //check local
      let csvDataString = localStorage.getItem(layer.id);
      //not available, fetch
      if (!csvDataString) {
        console.log("data not found in local storage, fetch");
        (async () => {
          // Your async code here
          fetchData(layer).then((data) => {
            plotString(data, layer);
          });
        })();
      } else {
        plotString(csvDataString, layer);
        console.log("load data from local storage, fetch");
      }
    };

    const togglelayer = (layer) => {
      let l = toRaw(layer);
      if (l.isactive) {
        l.leaflet.forEach((element) => {
          map.value.addLayer(element);
        });
      } else {
        l.leaflet.forEach((element) => {
          map.value.removeLayer(element);
        });
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

    return {
      fetchData,
      plotData,
      changeTileLayer,
      layers,
      togglelayer,
    };
  },
};
</script>

<style scoped>
#map {
  width: 100%;
  height: 500px;
}
</style>
