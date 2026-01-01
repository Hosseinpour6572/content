# Geometry Converter API

A lightweight REST API that wraps [`ogr2ogr`](https://gdal.org/programs/ogr2ogr.html) to convert geospatial vector files (DXF, DWG via DXF, DGN, GeoJSON, Shapefile, etc.) and optionally reproject them between geographic (lat/lon) and projected (e.g., UTM) coordinate systems.

> This project assumes `ogr2ogr` is available on the host system (for example via GDAL). The API reports a clear error if the binary cannot be found.

## Features
- Convert vector files between many formats supported by GDAL/OGR (DXF, DGN, GeoJSON, Shapefile, GPKG, KML, etc.).
- Reproject inputs using `-s_srs` and `-t_srs` (e.g., WGS84 ↔ UTM zones).
- JSON base64 and binary uploads with configurable file-size limits.
- Healthcheck endpoint for deployment monitoring.

## Quick start
1. Ensure `ogr2ogr` is installed and available on `PATH`.
2. Install dependencies and start the server (default port `5000`):

```bash
cd geometry-converter-api
npm install
npm start
```

## API

### `GET /health`
Simple liveness probe. Returns JSON `{ "status": "ok" }`.

### `POST /convert`
Converts an uploaded file to a target format and, optionally, reprojects coordinates. Two payload styles are supported:

- **JSON** (`application/json`):
  - `fileBase64` (required): Base64-encoded contents of the input file.
  - `targetFormat` (required): Target format name understood by `ogr2ogr` (e.g., `DXF`, `GeoJSON`, `GPKG`).
  - `sourceSrs` (optional): Source coordinate system (e.g., `EPSG:4326`).
  - `targetSrs` (optional): Target coordinate system (e.g., `EPSG:32638` for UTM zone 38N). If omitted, no reprojection is applied.
  - `fileName` (optional): Used for content-disposition naming.

- **Binary** (`application/octet-stream`):
  - Send the file body as the request payload.
  - Pass `targetFormat`, `sourceSrs`, and `targetSrs` via query parameters.
  - Optional `fileName` query parameter adjusts the download name.

- **Response:** Binary output of the converted file with a `Content-Disposition` header set to download the file.

- **Example (JSON payload): GeoJSON (WGS84) → DXF (UTM zone 38N)**

```bash
curl -X POST http://localhost:5000/convert \
  -H \"Content-Type: application/json\" \
  -d \"$(jq -n --arg data \"$(base64 -w0 /path/to/input.geojson)\" '{fileBase64:$data, targetFormat:\"DXF\", sourceSrs:\"EPSG:4326\", targetSrs:\"EPSG:32638\", fileName:\"input.geojson\"}')\" \
  --output output.dxf
```

### Supported formats
`ogr2ogr` supports many vector formats. Common options for this API:
- GeoJSON / GeoJSONSeq
- DXF (useful for DWG by first converting DWG→DXF with ODA/FME if needed)
- DGN
- ESRI Shapefile
- GeoPackage (GPKG)
- KML/KMZ

Use `ogrinfo --formats` on your system to list the installed drivers.

## Configuration
- `PORT`: Port to listen on (default `5000`).
- `MAX_UPLOAD_MB`: Maximum upload size in megabytes (default `50`).

## Notes on DWG
Direct DWG support depends on how GDAL was built. If your GDAL install lacks DWG write support, convert DWG→DXF first (using ODA File Converter, Teigha, or FME), then call this API to go DXF→target.

## Running tests
```bash
npm test
```

Tests use a stub converter and do not require `ogr2ogr` to be installed. If you prefer to avoid installing dependencies globally, run tests from the repository root where `express` is already available.
