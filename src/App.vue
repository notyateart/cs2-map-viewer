<template>
  <div class="flex h-screen flex-col">
    <!-- Header -->
    <header
      class="z-10 flex items-center justify-between bg-white p-4 text-base font-medium tracking-tight text-zinc-900 shadow-xl dark:bg-zinc-800 dark:text-white"
    >
      <h1 class="text-2xl">CS2 Map Viewer</h1>
      <div class="flex items-center space-x-2">
        <!-- GeoJSON upload -->
        <input
          id="multi-upload"
          type="file"
          multiple
          accept=".json"
          @change="handleFiles"
          class="hidden"
        />

        <!-- OSM upload -->
        <input
          id="osm-upload"
          type="file"
          accept=".osm,.xml"
          @change="handleOSM"
          class="hidden"
        />

        <button
          @click="triggerUpload"
          class="rounded bg-blue-500 px-4 py-2 text-white hover:bg-blue-600"
        >
          <i class="fa-solid fa-upload mr-2"></i>Load Json Files (Carto)
        </button>

        <button
          @click="triggerOSMUpload"
          class="rounded bg-purple-500 px-4 py-2 text-white hover:bg-purple-600"
        >
          <i class="fa-solid fa-file-code mr-2"></i>Load OSM File (OSM Export)
        </button>

        <button
          @click="toggleThemeMode"
          class="rounded border border-zinc-300 px-4 py-2 transition hover:bg-zinc-100 dark:border-zinc-600 dark:hover:bg-zinc-700"
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
    </header>

    <!-- Main layout -->
    <div class="flex flex-1">
      <!-- Sidebar -->
      <aside
        class="w-80 overflow-y-auto bg-white p-4 text-base font-medium tracking-tight text-zinc-900 shadow-xl dark:bg-zinc-800 dark:text-white"
      >
        <h2 class="mb-2 text-lg font-semibold">Layers</h2>
        <div v-for="(state, id) in layerStates" :key="id" class="mb-4">
          <div class="mb-1 flex items-center justify-between">
            <label class="font-medium capitalize">{{ id }}</label>
            <input type="checkbox" v-model="layerStates[id].visible" />
          </div>
          <input type="color" v-model="layerStates[id].color" class="w-full" />
        </div>
      </aside>

      <!-- Map -->
      <main class="relative flex-1">
        <MapView
          :layers="geojsonLayers"
          :layerStates="layerStates"
          :theme="themeMode"
        />
      </main>
    </div>
  </div>
</template>

<script setup>
import osmtogeojson from "osmtogeojson";
import { ref, onMounted } from "vue";
import MapView from "./components/MapView.vue";

const themeMode = ref("system");
const isDark = ref(false);

onMounted(() => {
  const stored = localStorage.theme;
  if (stored === "light" || stored === "dark") {
    themeMode.value = stored;
  }
  applyTheme();
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

const geojsonLayers = ref([]);
const layerStates = ref({});

function triggerUpload() {
  document.getElementById("multi-upload").click();
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

function stripZ(coords) {
  if (typeof coords[0] === "number") return coords.slice(0, 2);
  return coords.map(stripZ);
}

function triggerOSMUpload() {
  document.getElementById("osm-upload").click();
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

      for (const f of geojson.features) {
        const tags = f.properties || {};
        const coords = stripZ(f.geometry.coordinates);

        for (const [key, value] of Object.entries(tags)) {
          if (!value) continue;

          const layerId = `osm-${sanitize(key)}-${sanitize(value)}`;

          if (!layerMap[layerId]) {
            layerMap[layerId] = {
              type: "FeatureCollection",
              features: [],
            };
          }

          const cleanFeature = {
            ...f,
            geometry: {
              ...f.geometry,
              coordinates: coords,
            },
          };

          layerMap[layerId].features.push(cleanFeature);
        }
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

function sanitize(str) {
  return str.toLowerCase().replace(/[^a-z0-9_-]+/g, "-");
}

function getTagColor(id) {
  if (id.includes("building")) return "#10B981";
  if (id.includes("highway")) return "#F59E0B";
  if (id.includes("landuse")) return "#8B5CF6";
  if (id.includes("natural")) return "#22D3EE";
  if (id.includes("railway")) return "#EF4444";
  if (id.includes("power")) return "#6B7280";
  return (
    "#" +
    Math.floor(Math.random() * 0xffffff)
      .toString(16)
      .padStart(6, "0")
  );
}

function randomColor() {
  return (
    "#" +
    Math.floor(Math.random() * 0xffffff)
      .toString(16)
      .padStart(6, "0")
  );
}
</script>
