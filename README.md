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


## If no rst is availeb and you have a tif, use this##

{
 "cells": [
  {
   "cell_type": "markdown",
   "id": "e723e03c",
   "metadata": {},
   "source": [
    "# 1. After creating the stream link in Qgis, resample it to the DEM (reference )"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "e1f1774a",
   "metadata": {},
   "outputs": [],
   "source": [
    "import rasterio\n",
    "from rasterio.warp import reproject, Resampling\n",
    "from rasterio.enums import Resampling as RioResampling\n",
    "from rasterio.transform import from_origin\n",
    "import numpy as np\n",
    "import os\n",
    "\n",
    "# Input files\n",
    "streamlink_path = r\"E:\\DEMs\\Beamon_Creek_Boyer_River\\Streamlink.tif\"\n",
    "template_rst_path = r\"C:\\Users\\dplatero\\watem-sedem-master\\watem_sedem\\Beamon_Creek_Boyer_River\\Input\\Beamon_creek_fill.rst\"\n",
    "output_rst_path = r\"C:\\Users\\dplatero\\watem-sedem-master\\watem_sedem\\Beamon_Creek_Boyer_River\\Input\\StreamLink_aligned.rst\"\n",
    "\n",
    "# Open the template .rst (Beamon_creek_fill) to get georeferencing\n",
    "with rasterio.open(template_rst_path) as template:\n",
    "    template_meta = template.meta.copy()\n",
    "    template_crs = template.crs\n",
    "    template_transform = template.transform\n",
    "    template_shape = (template.height, template.width)\n",
    "\n",
    "# Open the streamlink raster\n",
    "with rasterio.open(streamlink_path) as src:\n",
    "    src_data = src.read(1)\n",
    "    src_transform = src.transform\n",
    "    src_crs = src.crs\n",
    "\n",
    "    # Prepare destination array with correct shape and dtype\n",
    "    destination = np.zeros(template_shape, dtype=np.int16)\n",
    "\n",
    "    # Reproject to align with template raster\n",
    "    reproject(\n",
    "        source=src_data,\n",
    "        destination=destination,\n",
    "        src_transform=src_transform,\n",
    "        src_crs=src_crs,\n",
    "        dst_transform=template_transform,\n",
    "        dst_crs=template_crs,\n",
    "        resampling=Resampling.nearest\n",
    "    )\n",
    "\n",
    "# Update metadata for output\n",
    "template_meta.update({\n",
    "    'driver': 'RST',\n",
    "    'dtype': 'int16',\n",
    "    'count': 1,\n",
    "    'nodata': 0  # use 0 if background is streamless\n",
    "})\n",
    "\n",
    "# Save as aligned .rst\n",
    "with rasterio.open(output_rst_path, 'w', **template_meta) as dst:\n",
    "    dst.write(destination, 1)\n",
    "\n",
    "print(\"✅ Aligned StreamLink raster saved to:\", output_rst_path)\n"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "57f31449",
   "metadata": {},
   "source": [
    "# 2. I like to check all rasters for their properties "
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "2ed7cef5",
   "metadata": {},
   "outputs": [],
   "source": [
    "import rasterio\n",
    "import os\n",
    "\n",
    "# List of raster paths\n",
    "raster_paths = [\n",
    "    r\"C:\\Users\\dplatero\\watem-sedem-master\\watem_sedem\\Beamon_Creek_Boyer_River\\Input\\StreamLink_aligned.rst\",\n",
    "    r\"C:\\Users\\dplatero\\watem-sedem-master\\watem_sedem\\Beamon_Creek_Boyer_River\\Input\\Beamon_creek_fill.rst\",\n",
    "    r\"C:\\Users\\dplatero\\watem-sedem-master\\watem_sedem\\Beamon_Creek_Boyer_River\\Input\\Beamon_Rfactor.rst\",\n",
    "    r\"C:\\Users\\dplatero\\watem-sedem-master\\watem_sedem\\Beamon_Creek_Boyer_River\\Input\\CFactor.rst\",\n",
    "    r\"C:\\Users\\dplatero\\watem-sedem-master\\watem_sedem\\Beamon_Creek_Boyer_River\\Input\\KFactor.rst\",\n",
    "    r\"C:\\Users\\dplatero\\watem-sedem-master\\watem_sedem\\Beamon_Creek_Boyer_River\\Input\\KTCmap.rst\",\n",
    "    r\"C:\\Users\\dplatero\\watem-sedem-master\\watem_sedem\\Beamon_Creek_Boyer_River\\Input\\parcel_map.rst\",\n",
    "    r\"C:\\Users\\dplatero\\watem-sedem-master\\watem_sedem\\Beamon_Creek_Boyer_River\\Input\\PFactor.rst\"\n",
    "]\n",
    "\n",
    "# Print summary\n",
    "for path in raster_paths:\n",
    "    with rasterio.open(path) as src:\n",
    "        print(f\"📄 {os.path.basename(path)}\")\n",
    "        print(f\"  - CRS:        {src.crs}\")\n",
    "        print(f\"  - Size:       {src.width} x {src.height} (cols x rows)\")\n",
    "        print(f\"  - Pixel size: {src.transform.a} x {-src.transform.e} (X x Y)\")\n",
    "        print(f\"  - Dtype:      {src.dtypes[0]}\")\n",
    "        print(f\"  - NoData:     {src.nodata}\")\n",
    "        print(\"-\" * 50)\n"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "7699ffff",
   "metadata": {},
   "source": [
    "# 3. Adjacent txt creation"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "6ca7e19b",
   "metadata": {},
   "outputs": [],
   "source": [
    "import rasterio\n",
    "import numpy as np\n",
    "import pandas as pd\n",
    "from collections import defaultdict\n",
    "\n",
    "# Load stream segment raster\n",
    "with rasterio.open(r\"C:\\Users\\dplatero\\watem-sedem-master\\watem_sedem\\Beamon_Creek_Boyer_River\\Input\\StreamLink_aligned.rst\") as src:\n",
    "    segments = src.read(1)\n",
    "    transform = src.transform\n",
    "\n",
    "rows, cols = segments.shape\n",
    "neighbors = [(-1,0), (1,0), (0,-1), (0,1), (-1,-1), (-1,1), (1,-1), (1,1)]\n",
    "connections = set()\n",
    "\n",
    "# Loop over raster pixels\n",
    "for row in range(rows):\n",
    "    for col in range(cols):\n",
    "        seg_id = segments[row, col]\n",
    "        if seg_id <= 0:\n",
    "            continue\n",
    "        for dr, dc in neighbors:\n",
    "            r, c = row + dr, col + dc\n",
    "            if 0 <= r < rows and 0 <= c < cols:\n",
    "                neighbor_id = segments[r, c]\n",
    "                if neighbor_id > 0 and neighbor_id != seg_id:\n",
    "                    connections.add((seg_id, neighbor_id))\n",
    "\n",
    "# Remove upstream duplicates (only retain downstream flow)\n",
    "# (this is optional and requires flow direction logic — can be added later)\n",
    "\n",
    "# Save to tab-separated file\n",
    "df = pd.DataFrame(list(connections), columns=[\"from\", \"to\"])\n",
    "df.to_csv(\"adjacent_segments.txt\", sep=\"\\t\", index=False)\n",
    "# ...existing code...\n",
    "output_path = r\"C:\\Users\\dplatero\\watem-sedem-master\\watem_sedem\\Beamon_Creek_Boyer_River\\Input\\adjacent_segments.txt\"\n",
    "df.to_csv(output_path, sep=\"\\t\", index=False)\n",
    "print(f\"Saved adjacent segments to: {output_path}\")"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "0a98a29b",
   "metadata": {},
   "source": [
    "# 4. Upstream txt creation"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "f6835ac6",
   "metadata": {},
   "outputs": [],
   "source": [
    "import pandas as pd\n",
    "\n",
    "# Path to previous adjacent segment file (from → to)\n",
    "adjacent_path = r\"C:\\Users\\dplatero\\watem-sedem-master\\watem_sedem\\Beamon_Creek_Boyer_River\\Input\\adjacent_segments.txt\"\n",
    "upstream_path = r\"C:\\Users\\dplatero\\watem-sedem-master\\watem_sedem\\Beamon_Creek_Boyer_River\\Input\\upstream_segments.txt\"\n",
    "\n",
    "# Load adjacent segments\n",
    "df_adj = pd.read_csv(adjacent_path, sep=\"\\t\")\n",
    "\n",
    "# Reverse direction: to = edge, from = upstream_edge\n",
    "df_up = df_adj.rename(columns={\"from\": \"upstream_edge\", \"to\": \"edge\"})\n",
    "\n",
    "# Add proportion = 1.0\n",
    "df_up[\"proportion\"] = 1.0\n",
    "\n",
    "# Reorder columns\n",
    "df_up = df_up[[\"edge\", \"upstream_edge\", \"proportion\"]]\n",
    "\n",
    "# Save to tab-delimited file\n",
    "df_up.to_csv(upstream_path, sep=\"\\t\", index=False)\n",
    "\n",
    "print(f\"✅ Upstream segment table written to:\\n{upstream_path}\")\n"
   ]
  },
  {
   "cell_type": "markdown",
   "id": "3a2e6748",
   "metadata": {},
   "source": [
    "# 5. River routing raster creation"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "id": "84d73891",
   "metadata": {},
   "outputs": [],
   "source": [
    "import rasterio\n",
    "import numpy as np\n",
    "\n",
    "# Load base stream raster to get shape and transform\n",
    "with rasterio.open(r\"C:\\Users\\dplatero\\watem-sedem-master\\watem_sedem\\Beamon_Creek_Boyer_River\\Input\\StreamLink_aligned.rst\") as src:\n",
    "    meta = src.meta.copy()\n",
    "    shape = src.shape\n",
    "\n",
    "# Create blank routing raster\n",
    "routing = np.zeros(shape, dtype=np.int16)\n",
    "\n",
    "# Example: impose routing value 6 at a certain location\n",
    "routing[50, 100] = 6  # row 50, col 100 = flow to lower-left\n",
    "\n",
    "# Save to disk at your desired location\n",
    "output_path = r\"C:\\Users\\dplatero\\watem-sedem-master\\watem_sedem\\Beamon_Creek_Boyer_River\\Input\\river_routing_map.rst\"\n",
    "meta.update(dtype='int16', nodata=0)\n",
    "with rasterio.open(output_path, \"w\", **meta) as dst:\n",
    "    dst.write(routing, 1)\n",
    "\n",
    "print(f\"Saved routing raster to: {output_path}\")"
   ]
  }
 ],
 "metadata": {
  "language_info": {
   "name": "python"
  }
 },
 "nbformat": 4,
 "nbformat_minor": 5
}

