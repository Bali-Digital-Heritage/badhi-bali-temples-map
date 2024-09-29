<template lang="pug">
.corner.p-5.pt-3
  .mb-6
    p.title Peta Pura Bali
    p.small  Persebaran pura yang terdata di Pulau Bali (tidak mewakili keseluruhan pura yang ada).
  template(v-if="!hidepanel")
    .legend.mt-3
#map
#footer 
  small
    | Developed by the 
    a(href="https://badhi.id", target="_blank") BADHI project 
    | team. Version September 2024.
    |
    br
    | Data source: temples data from the Temple Restore project run by Samatha Sharma, MSc, London School of Economics, B.E(Hons.) BITS Pilani. She can be reached at samatha.express@gmail.com
</template>

<style>
#footer {
  position: absolute;
  bottom: 2%;
  z-index: 1002;
  text-align: center;
  width: 100%;
}
#map {
  width: 100%;
  height: 500px;
}

.panel {
  max-width: 500px;
}

.popup {
  max-height: 600px;
  overflow-y: scroll;
}

.popup h3{
  font-size: 1.5rem;
  margin-bottom: 10px;
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
  left: 5%;
  max-width: 500px;
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
    const hidepanel = ref(false);

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

    const TemplesPointLayer = {
      type: LAYERTYPE.point,
      tokenurl: "https://badhi-data-api.netlify.app/api/token",
      csvurl: "https://badhi-data-api.netlify.app/api/data",
      name: "Temple Restore Data",
      description:
        "Temples data from the Temple Restore project run by Samatha Sharma, MSc, London School of Economics, B.E(Hons.) BITS Pilani. She can be reached at samatha.express@gmail.com",
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

    const layers = ref([TemplesPointLayer]);

    let tileLayer;
    onMounted(async () => {
      // Initialize the map
      var southWest = L.latLng(-9.089964, 114.101455);
      var northEast = L.latLng(-7.825840, 115.885725);
      var baliBounds = L.latLngBounds(southWest, northEast);
      map.value = L.map("map", {
        minZoom: 10,
        maxZoom: 18,
        maxBounds: baliBounds,
        maxBoundsViscosity: 1.0,
      }).setView([-8.369663, 115.190955], 10);

      L.tileLayer(
        "https://{s}.tile-cyclosm.openstreetmap.fr/cyclosm/{z}/{x}/{y}.png",
        {
          maxZoom: 20,
          attribution:
            '<a href="https://github.com/cyclosm/cyclosm-cartocss-style/releases" title="CyclOSM - Open Bicycle render">CyclOSM</a> | Map data: &copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
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
      //get token
      if (layer.tokenurl) {
        try {
          const response = await axios.get(layer.tokenurl);
          layer.token = response.data.token;
          try {
            const response = await axios.get(layer.csvurl, {
              headers: {
                Authorization: `Bearer ${layer.token}`,
              },
              responseType: "json",
            });
            csvData = {
              type: "json",
              data: response.data,
            };
          } catch (error) {
            console.error("Error fetching the CSV data", error);
          }
        } catch (error) {
          console.error("Error fetching token:", error);
        }
      } else {
        try {
          const response = await axios.get(layer.csvurl);
          csvData = {
            type: "blob",
            data: response.data,
          };
        } catch (error) {
          console.error("Error fetching the CSV data", error);
        }
      }

      return csvData;
    };
    const createPopupContent = (row) => {
      let table = `<div class='popup'>
        <h3><b>${row['Temple Name']}</b></h3>
        <table>`;
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

    const plotString = (csvData, layer) => {
      // Parse the CSV data
      layer.leaflet = [];

      let results = [];

      switch (csvData.type) {
        case "blob":
          Papa.parse(csvData.data, {
            header: true,
            complete: (res) => {
              results = res.data;
            },
          });
          break;
        case "json":
          results = csvData.data;
          break;
      }

      results.forEach((row) => {
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
    };

    const plotData = (layer) => {
      (async () => {
        // Your async code here
        fetchData(layer).then((data) => {
          plotString(data, layer);
        });
      })();
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
      hidepanel,
    };
  },
};
</script>
