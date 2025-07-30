# River-Segement-River-Routing-Creation-for-WaTEM-SEDEM

 WaTEM/SEDEM River Segment Preparation using QGIS + WhiteboxTools
This guide documents the step-by-step process for generating a river segment raster, stream network routing files, and routing maps for use in the WaTEM/SEDEM soil erosion and sediment delivery model using QGIS + WhiteboxTools plugin.

📦 Input Data
DEM (e.g., 10m resolution raster)

Pre-processed watershed mask (if clipped)

🔄 Workflow Overview

  A[Fill or Breach DEM] --> B[D8 Pointer]
  B --> C[D8 Flow Accumulation]
  C --> D[Extract Streams]
  D & B --> E[Stream Link Identifier]
  E --> F[River Segment Raster]
  F --> G[Upstream Segments Table]
  F --> H[Adjacent Segments Table]
  F --> I[Routing Map (Optional)]
🛠️ Step-by-Step Instructions
1. Hydrologic Conditioning
Use Breach Depressions Least Cost or Fill Depressions
→ Output: breached_dem.tif

2. Flow Direction
Run D8 Pointer using the breached DEM
→ Output: d8_pointer.tif

3. Flow Accumulation
Use D8 Flow Accumulation with:

Input: d8_pointer.tif

Check: Input is pointer

Check: Esri pointer (✅ if pointer was made in ArcGIS)
→ Output: flow_accum.tif

4. Extract Streams
Tool: Extract Streams

Input: flow_accum.tif

Threshold: 300.0 (adjust based on density)

Output: streams.tif (binary: 1 = stream, 0 = background)

5. Generate Stream Segments
Tool: Stream Link Identifier

Input D8 Pointer: d8_pointer.tif

Input Streams: streams.tif

Check: Use zero background

Output: stream_segments.tif (unique segment ID per link)

📁 WaTEM/SEDEM Files Generated
File	Purpose
stream_segments.tif → StreamLink_aligned.rst	River segment raster (Int16)
adjacent_segments.txt	Segment-to-segment connectivity (from → to)
upstream_segments.txt	Segment-to-segment input (to ← from) with proportions
river_routing_map.rst	Optional raster to enforce flow directions (0–8)

🧠 Notes on Segment Tables
➕ adjacent_segments.txt
Format: from<TAB>to

One line per connection (even if multiple upstreams feed same downstream)

🔁 upstream_segments.txt
Format: edge<TAB>upstream_edge<TAB>proportion

Proportion usually 1.0 unless split routing is needed

🔄 river_routing_map.rst
Format: raster with directional values 0–8

Use to manually enforce routing over culverts, dams, ditches, etc.

🗂️ Folder Structure Example
css
Copy
Edit
Beamon_Creek_Boyer_River/
├── Input/
│   ├── Beamon_creek_fill.rst
│   ├── StreamLink_aligned.rst
│   ├── upstream_segments.txt
│   ├── adjacent_segments.txt
│   ├── river_routing_map.rst
├── Output/
│   └── [WaTEM/SEDEM run outputs]
├── README.md
🧪 Tips
Use Raster → Miscellaneous → Align Rasters to match resolution, extent, CRS

Ensure all rasters are Int16 or Float32, with nodata set consistently

Document your threshold, flow direction scheme (Whitebox vs ESRI), and DEM resolution

