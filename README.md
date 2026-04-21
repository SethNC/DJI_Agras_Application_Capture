# DJI Agras Flight Data Recorder & Spray Drone Coverage Polygon Visualizer

A lightweight browser console script that captures per-second flight telemetry from the **DJI Agras SmartFarm web replay** page when official export options are limited or unavailable.

The script records GPS position, spray flow rate, route spacing, and task speed **only during active spraying** (filters out all lines where spray flow rate = 0). It also pulls static information (Team Name and Aircraft) and includes proper units in the CSV headers.

## Important Notes

**Timestamp Warning**:  
The `Timestamp` column uses your **system/browser time** (`new Date().toISOString()`), **not** the original GPS flight timestamp. The web replay does not expose the actual flight GPS timestamps in the visible DOM.

- The script only captures data that is visibly displayed on the playback page.
- Designed for the current DJI Agras web interface (as of 2026).

## Features

- Records data every second while the flight animation plays
- Automatically skips non-spraying segments (`spray_flow_rate = 0`)
- Includes units in CSV column headers (e.g., `Spray_Flow_Rate_L_per_min`)
- Swaps Team Name and Aircraft columns as requested
- Clean, timestamped CSV filename
- Live console logging for verification
- Easy start/stop via console commands

## How to Use

1. Open your flight replay in the DJI Agras / SmartFarm web portal.
2. Start playing the flight animation (normal speed or faster works fine).
3. Open Browser DevTools:
   - Press `F12` or `Ctrl + Shift + I` (Windows/Linux)
   - Or `Cmd + Option + I` (Mac)
4. Go to the **Console** tab.
5. Copy and paste the **entire script** below into the console and press **Enter**.

   The script will automatically start recording.

```javascript
// === DJI Agras Flight Data Recorder v4 - Perfected ===
// Features: Filters out spray_flow_rate = 0, Swapped Team/Aircraft columns, Units in headers
let dataLog = [];
let recordingInterval = null;

// === Capture static info once (titles, units, drone & team) ===
function captureStaticInfo() {
  const titles = Array.from(document.querySelectorAll('span._text-title_1cwbi_8')).map(el => el.textContent.trim());
  const values = Array.from(document.querySelectorAll('span._text-value_1cwbi_19')).map(el => el.textContent.trim());
  const units = Array.from(document.querySelectorAll('span._text-unit_1cwbi_16')).map(el => el.textContent.trim());
  const aircraftIndex = titles.indexOf('Aircraft');
  const teamIndex = titles.indexOf('Team Name');
  const staticData = {
    aircraft: aircraftIndex !== -1 ? values[aircraftIndex] || 'Maxine' : 'Maxine',
    teamName: teamIndex !== -1 ? values[teamIndex] || 'NCWolfpack' : 'NCWolfpack',
    unitFlow: units[0] || 'L/min',
    unitSpacing: units[1] || 'm',
    unitSpeed: units[2] || 'm/s'
  };
  console.log("%c📋 Static info captured:", "color: orange; font-weight: bold", staticData);
  return staticData;
}

function startRecording() {
  const staticInfo = captureStaticInfo();
 
  console.log("%c🚁 DJI Agras recorder STARTED – capturing every second (spray > 0 only)", "color: lime; font-weight: bold; font-size: 14px");
  recordingInterval = setInterval(() => {
    const timestamp = new Date().toISOString();
    // === GPS ===
    const gpsElement = document.querySelector('._leaflet-control-marker-coordinate_811c0_1');
    let latitude = 'N/A';
    let longitude = 'N/A';
    if (gpsElement) {
      const gpsSpans = gpsElement.querySelectorAll('span');
      if (gpsSpans.length >= 2) {
        latitude = gpsSpans[0].textContent.trim();
        longitude = gpsSpans[1].textContent.trim();
      }
    }
    // === Dynamic values ===
    const valueElements = document.querySelectorAll('span._text-value_1cwbi_19');
    let sprayFlow = 'N/A';
    let routeSpacing = 'N/A';
    let taskSpeed = 'N/A';
    if (valueElements.length >= 3) {
      sprayFlow = valueElements[0].textContent.trim();
      routeSpacing = valueElements[1].textContent.trim();
      taskSpeed = valueElements[2].textContent.trim();
    }
    // Skip this row if spray_flow_rate is 0
    if (sprayFlow === '0' || sprayFlow === '0.0' || sprayFlow === '0.00' || sprayFlow === '') {
      return; // skip non-spraying seconds
    }
    const row = {
      timestamp: timestamp,
      latitude: latitude,
      longitude: longitude,
      spray_flow_rate: sprayFlow,
      route_spacing: routeSpacing,
      task_speed: taskSpeed
    };
    dataLog.push(row);
    console.log(row);
  }, 1000);
}

function stopRecording() {
  if (recordingInterval) clearInterval(recordingInterval);
 
  if (dataLog.length === 0) {
    console.log("No data recorded (all lines had spray flow = 0).");
    return;
  }
  const staticInfo = captureStaticInfo();
  // CSV header with units + swapped Team_Name / Aircraft
  let csv = `Timestamp,Latitude,Longitude,Spray_Flow_Rate_${staticInfo.unitFlow.replace('/', '_per_')},Route_Spacing_${staticInfo.unitSpacing},Task_Speed_${staticInfo.unitSpeed},Team_Name,Aircraft\n`;
  dataLog.forEach(row => {
    csv += `${row.timestamp},${row.latitude},${row.longitude},${row.spray_flow_rate},${row.route_spacing},${row.task_speed},${staticInfo.teamName},${staticInfo.aircraft}\n`;
  });
  // Download
  const blob = new Blob([csv], { type: "text/csv;charset=utf-8;" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = `dji-agras-flight_${staticInfo.teamName}_${staticInfo.aircraft}_${new Date().toISOString().slice(0,19).replace(/:/g, '-')}.csv`;
  a.click();
  console.log(`%c✅ Recording stopped – ${dataLog.length} spraying seconds captured! CSV downloaded.`,
              "color: cyan; font-weight: bold; font-size: 14px");
 
  dataLog = [];
}
// ======================
// START RECORDING:
startRecording();
// To stop and download the CSV, type this in the console and press Enter:
// stopRecording()
```

## Usage Tips

- After pasting the script once, you can reuse the functions without repasting the whole thing.
  - Type startRecording() → press Enter to begin a new recording.
  - Type stopRecording() → press Enter to finish and download the CSV.

- Use the Up Arrow key in the console to quickly recall previous commands.
- Use Tab for auto-completion (e.g., type startR then press Tab).
- You can run stopRecording() even while the flight is still playing — it will safely stop and download what was captured.
- Recommended: Let the full mission (or at least the spraying portions) play through while recording.

## Output
The downloaded CSV contains these columns:

- Timestamp (system/browser time)
- Latitude
- Longitude
- Spray_Flow_Rate_L_per_min (or the unit shown on the page)
- Route_Spacing_m
- Task_Speed_m_per_s
- Team_Name
- Aircraft

## Limitations

Timestamp is system time, not the original GPS flight time.
Only captures data visible on the current web replay page.
Hidden telemetry (tank level, altitude, nozzle RPM, battery, true GPS timestamps, etc.) is not available with this method.

# Spray Drone Coverage Polygon Visualizer

A browser-based tool that converts **DJI Agras spray drone flight logs** (CSV format) into **realistic coverage polygons** with **Gallons Per Acre (GPA)** application rates for visualization on an interactive Leaflet map.

## Features

- Supports **up to 10 CSV files** at once (multiple aircraft or flights)
- Creates **coverage polygons** representing the actual sprayed swath (path + route spacing)
- Filters out short segments — **minimum spray block size: 4 feet** (~1.22 m)
- Calculates and displays **application rate in Gallons Per Acre (GPA)**
- Colors polygons based on GPA intensity (green = low, red = high)
- Exports a clean **GeoJSON** file with all data including `application_rate_gpa`
- Works entirely in the browser — no installation required

## How to Use

### 1. Save the Tool

1. Copy the entire HTML code from the previous message.
2. Paste it into a new file and save it as:
spray-drone-coverage-visualizer.html
text### 2. Open the Tool

- Double-click the saved `.html` file to open it in any modern web browser (Chrome, Edge, or Firefox recommended).
- The map will load centered near Lenoir, NC (you can pan and zoom freely).

### 3. Upload Your Flight Logs

1. Click the button **"📤 Select CSV Files (up to 10)"**
2. Select one or more of your DJI Agras CSV flight log files.
- You can select multiple files at once by holding `Ctrl` (Windows) or `Cmd` (Mac) while clicking.
3. Click the **"🚁 Process All Selected Files"** button that appears.

The tool will:
- Parse all selected CSV files
- Skip very short segments (< 4 feet)
- Generate coverage polygons
- Color them by application rate (GPA)
- Fit the map to your data

### 4. Explore the Results

- **Hover or click** any polygon to see detailed information:
- Aircraft name
- Team
- GPA (gallons per acre)
- Speed (m/s)
- Swath width (m)
- Flow rate (L/min)
- Source file

- Use the layer control in the top-right to switch between **Street Map** and **Satellite** view.

### 5. Download GeoJSON

- Click **"⬇️ Download GeoJSON"** to save the processed data.
- The downloaded file (`drone-coverage-gpa_....geojson`) can be opened in:
- QGIS
- ArcGIS
- Google Earth
- Any other GIS software that supports GeoJSON

## CSV Requirements

Your CSV files should contain these columns (exact names matter):

- `Timestamp`
- `Latitude`
- `Longitude`
- `Spray_Flow_Rate_L_per_min`
- `Route_Spacing_m`
- `Task_Speed_m/s`
- `Aircraft` (optional)
- `Team_Name` (optional)

## Output GeoJSON Properties

Each polygon feature includes:

- `application_rate_gpa` ← **Gallons per acre**
- `speed_m_s`
- `route_spacing_m`
- `spray_flow_l_per_min`
- `aircraft`
- `team`
- `timestamp`
- `source_file`

## Tips for Best Results

- Use files from the same flight or mission for best visual alignment.
- The 4-foot minimum length helps remove noise from turns and GPS jitter.
- For very large datasets, processing may take a few seconds.
- You can combine data from **Maxine**, **Jenny**, and other aircraft in one map.

## Troubleshooting

**Multiple files not working?**
- Make sure you click **"Process All Selected Files"** after selecting the files.
- Try selecting fewer files first to test.

**No polygons appear?**
- Check the browser console (F12 → Console tab) for messages.
- Ensure your CSV has valid Latitude/Longitude values.

**Map looks off?**
- Click "Clear Map" and try processing again.

## Technical Notes

- Built with **Leaflet.js** for mapping
- All calculations performed client-side (no data leaves your computer)
- GPA formula uses standard conversion: liters → gallons and m² → acres
- Polygon buffering approximates swath width using perpendicular offsets

---

**Created for NCWolfpack**  
For visualizing DJI Agras spray drone application coverage with accurate GPA rates.
