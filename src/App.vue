<template>
  <div class="flex h-screen flex-col">
    <!-- Header -->
    <nav class="z-10 bg-white shadow-lg dark:bg-gray-900">
      <div class="mx-auto flex flex-wrap items-center justify-between p-4">
        <a
          href="https://notyateart.github.io/cs2-map-viewer"
          class="flex items-center space-x-3 rtl:space-x-reverse"
        >
          <img src="./assets/cs2-map-viewer.svg" class="h-8" alt="Logo" />
          <span
            class="self-center text-2xl font-semibold whitespace-nowrap dark:text-white"
            >CS2 Map Viewer</span
          >
        </a>
        <div class="flex space-x-2">
          <input
            id="multi-upload"
            type="file"
            multiple
            accept=".json"
            @change="handleFiles"
            class="hidden"
          />
          <input
            id="osm-upload"
            type="file"
            accept=".osm,.xml"
            @change="handleOSM"
            class="hidden"
          />

          <button
            @click="triggerUpload"
            class="rounded border border-gray-300 px-4 py-2 text-gray-900 transition hover:bg-gray-100 dark:border-gray-600 dark:text-gray-50 dark:hover:bg-gray-700"
          >
            <i class="fa-solid fa-upload mr-2"></i>Load Carto (GeoJSON)
          </button>

          <button
            @click="triggerOSMUpload"
            class="rounded border border-gray-300 px-4 py-2 text-gray-900 transition hover:bg-gray-100 dark:border-gray-600 dark:text-gray-50 dark:hover:bg-gray-700"
          >
            <i class="fa-solid fa-upload mr-2"></i>Load OSM Export (OSM)
          </button>

          <button
            @click="toggleThemeMode"
            class="rounded border border-gray-300 px-4 py-2 text-gray-900 transition hover:bg-gray-100 dark:border-gray-600 dark:text-gray-50 dark:hover:bg-gray-700"
          >
            <i
              :class="{
                'fa-solid fa-sun': themeMode === 'light',
                'fa-solid fa-moon': themeMode === 'dark',
                'fa-solid fa-desktop': themeMode === 'system',
              }"
            ></i>
          </button>
        </div>
      </div>
    </nav>

    <!-- Main layout -->
    <div class="flex h-screen flex-1 overflow-hidden">
      <!-- Sidebar -->
      <aside
        class="flex w-80 flex-col overflow-y-scroll bg-white p-4 text-gray-900 shadow-xl dark:bg-gray-800 dark:text-white"
      >
        <h2 class="mb-3 text-lg font-semibold">Layers</h2>

        <div class="flex-1 pr-1">
          <div
            v-for="(state, id) in layerStates"
            :key="id"
            class="relative mb-2 flex items-center gap-3 rounded-sm border border-gray-200 px-3 py-2 dark:border-gray-700"
          >
            <input
              :id="`layer-toggle-${id}`"
              type="checkbox"
              v-model="layerStates[id].visible"
              class="h-4 w-4 rounded-sm border-gray-300 bg-gray-100 text-blue-600 focus:ring-2 focus:ring-blue-500 dark:border-gray-600 dark:bg-gray-700 dark:ring-offset-gray-800 dark:focus:ring-blue-600"
            />
            <label
              :for="`layer-toggle-${id}`"
              class="w-full text-sm font-medium text-gray-900 dark:text-gray-300"
            >
              {{ id }}
            </label>

            <!-- Color Circle + Picker -->
            <div class="relative" ref="colorPickerRefs">
              <!-- Trigger circle -->
              <div
                class="h-4 w-4 shrink-0 cursor-pointer rounded-full border border-gray-300"
                :style="{ backgroundColor: state.color }"
                @click="toggleColorPicker(id)"
              ></div>

              <!-- Color Picker Popover -->
              <div
                v-if="activeColorPicker === id"
                ref="activePopover"
                class="absolute right-0 z-50 mt-2 w-60 rounded-md border border-gray-300 bg-white p-2 shadow-md transition duration-150 dark:border-gray-700 dark:bg-gray-800"
              >
                <!-- Predefined color swatches -->
                <h6 class="mb-2">Swatches:</h6>
                <div class="mb-2 grid grid-cols-8 gap-2">
                  <div
                    v-for="color in predefinedColors"
                    :key="color"
                    class="h-5 w-5 cursor-pointer rounded-full border border-gray-200 hover:ring-2 hover:ring-offset-1"
                    :style="{ backgroundColor: color }"
                    @click="selectColor(id, color)"
                  ></div>
                </div>
                <!-- Custom color input -->
                 <h6 class="mt-6 mb-2">Cutom Color:</h6>
                <input
                  type="color"
                  v-model="layerStates[id].color"
                  class="h-8 w-full cursor-pointer rounded-sm border dark:border-gray-600 dark:bg-gray-700"
                />
              </div>
            </div>
          </div>
        </div>
      </aside>

      <!-- Map -->
      <main class="flex-1">
        <MapView
          :layers="geojsonLayers"
          :layerStates="layerStates"
        />
      </main>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from "vue";
import osmtogeojson from "osmtogeojson";
import MapView from "./components/MapView.vue";

// —————————————————————
// Theme & UI State
// —————————————————————
const themeMode = ref("system");
const isDark = ref(false);

// —————————————————————
// Layer Data & Visibility
// —————————————————————
const geojsonLayers = ref([]);
const layerStates = ref({
  background: {
    visible: true,
    color: "#364f37",
  },
});

// —————————————————————
// Color Picker State
// —————————————————————
const activeColorPicker = ref(null);
const activePopover = ref(null);
const predefinedColors = [
  "#ffb0be","#eb7289","#b7445d","#840e35","#ecc1c6","#bb9297","#8c666b","#5f3c41",
  "#ffb87e","#e48233","#b15300","#773000","#e9c6af","#b89781","#896a56","#5d402d",
  "#e5cf4b","#b59f00","#847200","#554800","#d5cfaa","#a5a07c","#787350","#4e4828",
  "#8be78f","#5bb661","#278733","#005a00","#bad7ba","#8ca78c","#607a60","#364f37",
  "#00ece3","#00b9b2","#008680","#005652","#a9d9d5","#7aa9a5","#4e7b78","#23504e",
  "#7fdaff","#00abed","#007cb7","#004f7a","#afd4ea","#81a4b9","#55778b","#2b4c5f",
  "#c1c7ff","#8d92f9","#6363c6","#3c3695","#c7cbef","#989cbe","#6b6f8f","#424562",
  "#ffaaff","#cc7bd1","#9c4ea1","#6d2073","#dfc3e0","#af94b0","#816882","#563e57",
  "#cecece","#9e9e9e","#717171","#484848"
];

function getTagColor(id) {
  if (id.includes("building")) return "#23504e";
  if (id.includes("highway")) return "#e48233";
  if (id.includes("landuse-farmland")) return "#b59f00";
  if (id.includes("landuse-landfill")) return "#787350";
  if (id.includes("landuse-forest")) return "#607a60";
  if (id.includes("landuse-industrial")) return "#8d92f9";
  if (id.includes("landuse-commercial")) return "#00abed";
  if (id.includes("landuse-residential")) return "#5bb661";
  if (id.includes("railway")) return "#cc7bd1";
  if (id.includes("natural")) return "#7fdaff";
  if (id.includes("power")) return "#d5cfaa";
  if (id.includes("amenity")) return "#5f3c41";
  return "#484848";
}

// —————————————————————
// Color Picker Functions
// —————————————————————
function toggleColorPicker(id) {
  activeColorPicker.value = activeColorPicker.value === id ? null : id;
}

function selectColor(id, color) {
  layerStates.value[id].color = color;
  activeColorPicker.value = null;
}

function handleClickOutside(event) {
  const popover = activePopover.value;
  if (popover && !popover.contains(event.target)) {
    activeColorPicker.value = null;
  }
}

function handleKeydown(event) {
  if (event.key === "Escape") {
    activeColorPicker.value = null;
  }
}

// —————————————————————
// Theme Handling
// —————————————————————
onMounted(() => {
  document.addEventListener("click", handleClickOutside);
  document.addEventListener("keydown", handleKeydown);

  const stored = localStorage.theme;
  if (stored === "light" || stored === "dark") themeMode.value = stored;
  applyTheme();
});

onBeforeUnmount(() => {
  document.removeEventListener("click", handleClickOutside);
  document.removeEventListener("keydown", handleKeydown);
});

function toggleThemeMode() {
  if (themeMode.value === "light") {
    themeMode.value = "dark";
    localStorage.theme = "dark";
  } else if (themeMode.value === "dark") {
    themeMode.value = "system";
    localStorage.removeItem("theme");
  } else {
    themeMode.value = "light";
    localStorage.theme = "light";
  }
  applyTheme();
}

function applyTheme() {
  const prefersDark = window.matchMedia("(prefers-color-scheme: dark)").matches;
  const stored = localStorage.theme;
  const shouldUseDark = stored === "dark" || (!stored && prefersDark);
  document.documentElement.classList.toggle("dark", shouldUseDark);
  isDark.value = shouldUseDark;
}

// —————————————————————
// Upload Handlers
// —————————————————————
function triggerUpload() {
  document.getElementById("multi-upload").click();
}

function triggerOSMUpload() {
  document.getElementById("osm-upload").click();
}

function handleFiles(event) {
  const selected = Array.from(event.target.files);
  const promises = selected.map((file) => {
    return new Promise((resolve) => {
      const reader = new FileReader();
      reader.onload = () => {
        try {
          const json = JSON.parse(reader.result);
          const name = file.name.split(".")[0];
          json.features = json.features.map((f) => ({
            ...f,
            geometry: {
              ...f.geometry,
              coordinates: stripZ(f.geometry.coordinates),
            },
          }));
          geojsonLayers.value.push({ id: name, data: json });
          layerStates.value[name] = {
            visible: true,
            color: randomColor(),
          };
        } catch (err) {
          console.error(`Error parsing ${file.name}`, err);
        } finally {
          resolve();
        }
      };
      reader.readAsText(file);
    });
  });

  Promise.all(promises);
}

function handleOSM(event) {
  const file = event.target.files[0];
  if (!file) return;

  const reader = new FileReader();
  reader.onload = () => {
    try {
      const xml = new DOMParser().parseFromString(reader.result, "text/xml");
      const geojson = osmtogeojson(xml);

      const layerMap = {};
      const primaryKeys = [
        "building", "highway", "landuse", "railway",
        "natural", "power", "amenity", "man_made",
      ];

      for (const f of geojson.features) {
        const tags = f.properties || {};
        const primaryKey = primaryKeys.find((k) => tags[k]);
        if (!primaryKey) continue;

        const value = sanitize(tags[primaryKey]);
        if (!value) continue;

        const layerId = `osm-${primaryKey}-${value}`;
        const coords = stripZ(f.geometry.coordinates);

        const cleanedFeature = {
          ...f,
          geometry: {
            ...f.geometry,
            coordinates: coords,
          },
        };

        if (!layerMap[layerId]) {
          layerMap[layerId] = { type: "FeatureCollection", features: [] };
        }

        layerMap[layerId].features.push(cleanedFeature);
      }

      for (const [id, data] of Object.entries(layerMap)) {
        if (data.features.length === 0) continue;
        geojsonLayers.value.push({ id, data });

        if (!layerStates.value[id]) {
          layerStates.value[id] = {
            visible: true,
            color: getTagColor(id),
          };
        }
      }
    } catch (err) {
      console.error("❌ Failed to parse .osm file:", err);
    }
  };

  reader.readAsText(file);
}

// —————————————————————
// Utilities
// —————————————————————
function sanitize(text) {
  return text.toLowerCase().replace(/[^a-z0-9_-]+/g, "-");
}

function stripZ(coords) {
  if (typeof coords[0] === "number") return coords.slice(0, 2);
  return coords.map(stripZ);
}

function randomColor() {
  return "#" + Math.floor(Math.random() * 0xffffff).toString(16).padStart(6, "0");
}
</script>

