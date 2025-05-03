# CS2 Map Viewer

![CS2 Map Viewer](readme-logo.png)

**[CS2 Map Viewer](https://notyateart.github.io/cs2-map-viewer/)** is a web-based tool for exploring and visualizing exported map data from **Cities: Skylines II**.

It supports both:
- **GeoJson files** exported via the [Carto](https://mods.paradoxplaza.com/mods/87428/Windows) mod (e.g. `Area.json`, `Building.json`, `Net_CenterLine.json`, etc.)
- **OSM files** exported via the [OSM Export](https://mods.paradoxplaza.com/mods/87422/Windows) mod (e.g. `export.osm`)


## What It Does

- Upload one or multiple `.json` or `.osm` files
- Automatically organizes layers based on content and tags (e.g. `landuse`, `building`, `power`)
- Lets you toggle visibility and customize colors for each layer


## How to use

### Export from CS2

With either [Carto](https://mods.paradoxplaza.com/mods/87428/Windows) or [OSM Export](https://mods.paradoxplaza.com/mods/87422/Windows) or both installed load a save game. Then:

1. Go to _Options_ > _OSM Export_ or _Carto_
2. Carto Users have to select GeoJson
3. Export
4. Wait a few minutes
5. The files are stored in `C:\Users\USERNAME\AppData\LocalLow\Colossal Order\Cities Skylines II\ModsData\Carto` or `...\OSMExport`


### Upload

Go to [notyateart.github.io/cs2-map-viewer](https://notyateart.github.io/cs2-map-viewer/)

| Source     | File Type | Notes |
|------------|-----------|-------|
| Carto      | `.json`   | Files like `Area.json`, `Building.json`, `Net_Edge.json`, etc. |
| OSM Export | `.osm`    | Must be valid OSM XML files exported from Cities: Skylines II |

You can upload both types in a single session — the tool will handle parsing and styling automatically.
This allows you to combine both exports. For example to load the zoning grid from Carto and the infrastructure from OSM Export

---

## Local Development

Install node.js from [https://nodejs.org](https://nodejs.org)

```bash
# Install yarn
npm install --global yarn
```

```bash
# Install Dependencies
yarn install
```