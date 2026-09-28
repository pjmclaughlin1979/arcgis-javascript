# City Quays 3 — ArcGIS Indoors model

An indoor model of City Quays 3 (92 Donegall Quay, Belfast), generated from the
RPP Architects general arrangement plans `CQ3_FP.pdf` (drawing series
`2403-RPP-01-ZZ-DR-A-2xx`, scale 1:100).

* `index.html` — 3D viewer (ArcGIS Maps SDK for JavaScript 5.1) with a floor filter,
  rooms coloured by use, extruded walls and a glass shell for the other floors.
  Serve the folder over HTTP (for example `python -m http.server`) and open
  `http://localhost:8000/cq3-indoors/`; opening the file directly will not load the data.
* `data/*.geojson` — the model, one file per ArcGIS Indoors layer:

| File | Geometry | Contents |
| --- | --- | --- |
| `Sites.geojson` | polygon | City Quays site (`SITE_ID`) |
| `Facilities.geojson` | polygon | the building (`FACILITY_ID` = `CQ3`) |
| `Levels.geojson` | polygon | 16 floorplates, `LEVEL_ID`, `LEVEL_NUMBER`, `VERTICAL_ORDER`, `ELEVATION_RELATIVE`, `HEIGHT_RELATIVE` |
| `Units.geojson` | polygon | rooms, cores, offices and terraces with `NAME`, `USE_TYPE`, `ROOM_NUMBER`, `LEVEL_ID` and an area check |
| `Details.geojson` | line | wall faces (`USE_TYPE` = `Wall`) |

Field names follow the ArcGIS Indoors Information Model (Facilities, Levels,
Units, Details with `FACILITY_ID` / `LEVEL_ID` keys), so the layers work as
floor-aware layers in ArcGIS Pro, Map Viewer and the JavaScript SDK.

## Loading into ArcGIS Pro / ArcGIS Indoors

1. Run **Create Indoors Database** (Indoors toolbox) to make an empty Indoors
   geodatabase in WGS 1984 or your project's coordinate system.
2. Run **JSON To Features** on each GeoJSON file.
3. Use **Append** (field mapping by name) to load them into the matching
   Indoors feature classes: Sites, Facilities, Levels, Units, Details.
4. Set the map's floor-awareness (Map Properties → Floors) to the Sites,
   Facilities and Levels layers if Pro does not detect it automatically.

The GeoJSON is 2D (WGS84) with elevations stored as attributes
(`ELEVATION_RELATIVE`, metres above ground floor), which is how Indoors stores
vertical position.

## How it was made

1. **Scale and grid.** The plans are vector CAD at 1:100 (1 pt = 35.28 mm).
   The structural grid bubbles give the origin (A/01) and confirm the scale:
   bays measure 3.80 m and 6.00 m, matching the dimension strings
   (3800 + 7 × 6000 + 3800 × 2250 / 9040 / 9020 / 1690 / 6850 / 500 / 2250 mm).
2. **Walls.** Wall lines are separated from hatching, fixtures, tags and grid
   lines by line weight, dash pattern, orientation and length.
3. **Rooms.** Every room tag (number, name, area) is read from the PDF text.
   Each room is flood-filled from its tag inside the wall raster (2 cm pixels),
   trying several wall sets and door-closing radii and keeping the result whose
   area best matches the printed area. The two stairs and the lift lobby use
   exact wall-face rectangles instead (stair treads and open lobby ends defeat
   the flood fill).
4. **Checks.** Each unit carries `AREA_PLAN_M2` (printed), `AREA_MODEL_M2`
   (measured) and `AREA_DIFF_PCT`.
5. **Georeferencing.** The sheet's north arrow puts sheet-up at a bearing of
   35.5°. The building centre is placed at an approximate position (see below).

## Assumptions and limits

* **Location is approximate.** No survey coordinates were available; the
  building centre is set to 54.6040° N, 5.9205° W (Clarendon Dock area), which
  may be tens of metres out. To correct it, change `ANCHOR` (and `BEARING` if
  needed) in `tools/build.py` and rebuild, or move the features in ArcGIS Pro.
* **Levels not on the drawings are inferred.** Sheets exist for Ground, 01, 04,
  06, 08, 09, 13, 14 and 15. Levels 02, 03, 05, 07 and 10 copy the nearest
  typical floor; levels 11 and 12 are LV09 less the "LV 11 terrace" strip shown
  on the upper-floor sheets. `SOURCE_NOTE` records this for every level and unit.
* **Heights are assumed.** No sections were supplied: ground floor 5.0 m,
  typical floors 3.9 m floor-to-floor. The building is reported as 70.3 m tall
  including rooftop plant.
* **Shell and core only.** The plans note that office fit-out is by others, so
  office floors are single open-plan units.
* Terraces are derived as the floorplate below minus the floorplate above.
* The LV13 Wellbeing Centre has no area tag; its outline is flood-filled only.
