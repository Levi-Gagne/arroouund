# World-Scale, Time-Aware 3D Map — Literature & Systems Survey

> A grounded survey of the prior art, standards, formats, tooling, open problems, and
> plausible architecture families for a spatially indexed model of the real world that can
> be queried by **place** and **time**.
>
> Scope note: this is a survey, not an architecture proposal. It separates what is already
> commodity from what is difficult-but-tractable engineering from what is still open research.
> Rendering is treated as downstream of the data model. Claims are cited inline; proprietary
> or unverified claims are flagged as such.

---

## 1. Executive Summary

The concept — capture parts of the world (LiDAR and/or imagery), attach them to a location
and time, merge them into a larger model, and serve back the *right representation for a place
right now or in the past* — is **not one problem. It is roughly seven**, and they are at very
different stages of maturity:

1. **Partitioning the Earth into addressable cells** — *commodity.* Mature, battle-tested
   global indexing systems already exist (Google **S2**, Uber **H3**, Geohash, Web-Mercator
   slippy/quadkey tiles, Plus Codes, MGRS). You should not invent a scheme. A cube-based
   intuition is essentially S2, which already exists and is in production at Google.
2. **Storing & streaming massive 3D geometry with level-of-detail** — *commodity / solved
   engineering.* Two OGC standards (**3D Tiles**, **I3S**) and cloud-optimized point-cloud
   formats (**COPC**, **EPT**) do this. Google serves its entire photorealistic world mesh as
   OGC 3D Tiles.
3. **Capturing the world with consumer devices** — *commodity / deployed.* iPhone/iPad LiDAR
   via ARKit, plus apps like Polycam, Scaniverse, Record3D, and Apple RoomPlan, already export
   usable point clouds, meshes, and even Gaussian splats.
4. **Localizing a device into a global frame (VPS)** — *difficult but deployed.* Google's
   ARCore Geospatial API and Niantic's VPS are live in ~90 countries, but the internals are
   proprietary and accuracy is scene-dependent.
5. **Separating geometry / appearance / semantics into layers** — *partially solved.* This is
   how mature systems are already structured (e.g. OpenUSD layering, CityGML semantic LODs,
   3D-Tiles metadata), so the instinct is sound — but no standard unifies all three at world
   scale.
6. **Versioning spatial data over time with multi-contributor merges** — *solved for 2D vector
   data; open for dense 3D.* OpenStreetMap, SQL:2011 temporal tables, and "geo-git" tools
   (Kart) handle versioned *vector* data well. There is **no standard for versioning dense 3D
   captures** (point clouds, meshes, NeRFs, splats).
7. **A genuinely time-aware, world-scale, crowd-updated 3D map** — *open research / very hard
   systems work.* Freshness/confidence scoring, cross-capture 3D change detection without
   drift, conflict resolution for merged captures, and streaming/LOD economics for 4D radiance
   fields are all unsolved at scale. **No surveyed system combines world-scale 3D geometry with
   first-class temporal versioning.** That specific combination is the open gap your idea
   lands in.

**Bottom line:** the *plumbing* (indexing, formats, tiling, capture, storage engines) is
largely off-the-shelf. The *hard, differentiated, and partly-unsolved* parts are time-versioning
of 3D, merging uneven multi-contributor captures, and freshness/quality governance.

---

## 2. Literature & Ecosystem Review (by category)

### 2.1 Global "photorealistic mesh" maps
- **Google Photorealistic 3D Tiles** is the largest commercial world-scale 3D mesh and is
  served in the **OGC 3D Tiles format** with glTF/Draco payloads — consumable by CesiumJS,
  deck.gl, Cesium for Unreal/Unity, etc. This is the strongest single signal that the industry
  has converged on an open streaming standard.
  <https://developers.google.com/maps/documentation/tile/3d-tiles-overview>
- **Google Immersive View** (places) was originally built on **NeRF**; Immersive View for
  Routes uses photogrammetric stitching of billions of images.
  <https://blog.google/products-and-platforms/products/maps/google-maps-immersive-view-routes/>
- **Cesium** created 3D Tiles and offers **CesiumJS** (open, Apache-2.0) plus **Cesium ion**, a
  proprietary "tiling-as-a-service" that ingests raw point clouds / photogrammetry / BIM and
  emits 3D Tiles. <https://cesium.com/platform/cesium-ion/3d-tiling-pipeline/>

### 2.2 Localization-first stacks (VPS)
These invert the problem: rather than serve geometry, they build a global feature map and
localize a camera against it.
- **Google ARCore Geospatial API / VPS** builds a global 3D point cloud from "tens of billions"
  of Street View images, anchoring by lat/long/altitude in ~90 countries. Documented accuracy
  is typically **< 5 m positional (often ~1 m), < 5° rotational** — but accuracy is
  scene-dependent, not a guarantee.
  <https://developers.google.com/ar/develop/geospatial>
- **Niantic VPS / "Large Geospatial Model" (LGM, Nov 2024)** claims to encode locations
  *implicitly in neural-network weights* rather than explicit geometry, built from crowdsourced
  scans. The headline figures (tens of billions of images, millions of scanned locations) are
  **marketing-scale and externally unverifiable**; LGM was announced as direction, not a shipped
  product. Niantic Spatial Inc. was spun out in March 2025.
  <https://nianticlabs.com/news/largegeospatialmodel>

### 2.3 Semantic / digital-twin stacks
Model the world as a graph of entities or a composed scene rather than a tiled mesh.
- **Microsoft Azure Digital Twins** uses **DTDL** (JSON-LD) to model a twin graph of
  properties/relationships/components.
  <https://learn.microsoft.com/en-us/azure/digital-twins/concepts-models>
- **NVIDIA Omniverse** uses **OpenUSD + RTX** as the substrate for industrial twins. (NVIDIA
  **Earth-2** is a *weather/climate forecasting* stack, not a geometric twin — a common point
  of confusion.) <https://www.nvidia.com/en-us/omniverse/>
- **Bentley iTwin** models infrastructure twins with an **immutable, append-only Changeset
  ledger** — the strongest *native geometric version control* of any surveyed platform.
  <https://www.itwinjs.org/learning/imodelhub/>
- **Esri ArcGIS** streams 3D via **I3S / Scene Layers**, the OGC counterpart to 3D Tiles.
  <https://docs.ogc.org/cs/17-014r9/17-014r9.html>

### 2.4 Reality capture (indoor & outdoor)
- **Matterport** hosts indoor digital twins on Matterport Cloud; **NavVis** and **Hexagon/Leica**
  serve professional reality-capture clouds (accuracy specs are largely proprietary/marketing).
  <https://matterport.com/news/matterport-reinvents-digital-twin-revolutionary-pro3-camera-and-new-cloud-platform>
- **Apple RoomPlan** outputs a *parametric* room model (USD/USDZ) — walls/doors/windows +
  16 object categories — i.e. **reconstructed semantics, not raw geometry**, with reported ~95%
  precision/recall on walls/windows. <https://machinelearning.apple.com/research/roomplan>
- **Consumer LiDAR**: iPhone/iPad Pro expose depth + `ARPointCloud` via ARKit's scene-depth API;
  **Polycam** exports 12+ formats (OBJ/FBX/glTF/point clouds/floorplans); **Scaniverse**
  (Niantic) supports Gaussian Splatting and exports the open **SPZ** format (reported ~90% size
  reduction — corroborated via search, worth direct confirmation if load-bearing).
  <https://developer.apple.com/documentation/ARKit/displaying-a-point-cloud-using-scene-depth> ·
  <https://learn.poly.cam/hc/en-us/articles/27756102599572-What-File-Types-Can-Polycam-Export>

### 2.5 Open / crowdsourced map data
- **OpenStreetMap** — per-object `version` + timestamp + changeset; a **weekly full-history
  planet file** preserves every version (incl. deletions) back to Oct 2007. The reference model
  for versioned, multi-contributor *vector* geodata. <https://wiki.openstreetmap.org/wiki/Planet.osm/full>
- **Overture Maps Foundation** — **GERS** provides stable persistent entity IDs across releases
  with a registry + bridge files for entity continuity over time; ships as cloud-native
  GeoParquet (~monthly). (A June 2025 UUID switch caused 100% ID churn — still maturing.)
  <https://docs.overturemaps.org/gers/>
- **Mapillary** (Meta, 2B+ images) and **KartaView** (Grab, 384M+ images) host crowdsourced
  street-level imagery that connects images across time and space.
  <https://www.mapillary.com/about>
- **Microsoft GlobalMLBuildingFootprints** — ~1.4B ML-extracted footprints (CDLA-Permissive),
  ~174M with heights. <https://github.com/microsoft/GlobalMLBuildingFootprints/>
- **3DCityDB** stores semantic CityGML models (LOD0–LOD4) in PostGIS/Oracle but has **no
  built-in versioning** — a notable temporal gap. <https://3dcitydb-docs.readthedocs.io/en/latest/overview/main-features.html>

### 2.6 The modern capture/representation research wave
- **NeRF** (2020) stores a scene as a small neural net (tens of MB) but renders slowly.
  <https://arxiv.org/abs/2210.00379>
- **3D Gaussian Splatting** (Kerbl et al., 2023) renders real-time (≥30 fps, 1080p) using
  millions of explicit anisotropic Gaussians (~1 GB; roughly 10–100× NeRF's storage) — it
  displaced NeRF for interactive use. <https://arxiv.org/abs/2308.04079>
- **City/world scale is an active 2024–2025 frontier, not solved:** Block-NeRF rendered a
  San-Francisco neighborhood from 2.8M images by decomposing into per-block NeRFs (explicitly
  appearance/time-robust across months); large-scale 3DGS (CityGaussianV2, BlockGaussian, LODGE)
  needs explicit block-partitioning + LOD. <https://waymo.com/research/block-nerf/> ·
  <https://arxiv.org/pdf/2411.00771>
- **Time/4D** extensions exist in research (**4D Gaussian Splatting**, ~82 fps dynamic scenes)
  but are single-scene, not world-scale. <https://arxiv.org/pdf/2310.08528>

---

## 3. Standards, Formats & Indexing Systems

### 3.1 Global spatial indexing — there *is* a mature default; don't invent one
There is no single universal winner, but there are clear pragmatic defaults by job. Notably,
**none of these natively encode altitude/time** — that is bolted on.

| System | Shape / basis | Hierarchy | Best for | Note |
|---|---|---|---|---|
| **Google S2** | Sphere → **cube** faces → quadtree, Hilbert curve | 31 levels (0–30); leaf ≈ 1 cm | Global geofencing, point/region queries in KV stores | 64-bit cell IDs; exact parent/child containment. **This is the "cube" intuition, already built.** |
| **Uber H3** | Icosahedron → hexagons (+12 pentagons) | 16 res (0–15); res-15 < 1 m² | Equal-area aggregation/analytics | Hex neighbors equidistant; parent/child containment only *approximate* geographically |
| **Geohash** | Interleaved lat/long bits, base-32 string | prefix = precision | Compact text keys, prefix proximity | Simple; rectangular cells distort by latitude |
| **Slippy XYZ / quadkey** | Web Mercator (EPSG:3857) | zoom = quadkey length | **Web map rendering/tiling — de-facto standard** | quadkey is prefix-hierarchical |
| **Plus Codes / OLC** | Lat/long base-20 grid | 10 chars ≈ 14 m, 11 ≈ 3 m | Human-facing addressing | Google, 2014 |
| **MGRS / UTM** | UTM zones → 100 km squares | down to 1 m | Military/surveying | 10 digits = 1 m |
| **OGC DGGS (Topic 21)** | Abstract spec for any conformant DGGS | multi-resolution + spatio-temporal | Standards anchor | Defines what a DGGS must do |
| **OGC GeoPose 1.0** | 6-DoF pose (position **+ orientation, incl. elevation**) | — | AR/VR & twin pose interop | Closest standard to 3D/indoor; has time-series structures |

Sources: S2 <https://s2geometry.io/devguide/s2cell_hierarchy.html> · H3 <https://h3geo.org/docs/core-library/overview/>
· Geohash <https://en.wikipedia.org/wiki/Geohash> · Slippy/quadkey
<https://learn.microsoft.com/en-us/bingmaps/articles/bing-maps-tile-system> · OLC
<https://github.com/google/open-location-code/blob/main/Documentation/Specification/specification.md>
· MGRS <https://en.wikipedia.org/wiki/Military_Grid_Reference_System> · OGC DGGS
<https://docs.ogc.org/as/20-040r3/20-040r3.html> · GeoPose <https://docs.ogc.org/is/21-056r11/21-056r11.html>

> **Sub-room / indoor / volumetric global addressing is the genuine gap.** S2/H3/Geohash/OLC/MGRS
> are 2D surface partitions. 3D is achieved by *pairing* a 2D cell with a separate altitude/floor
> dimension or by voxelizing point clouds. **GeoPose** is the most relevant standard (full 6-DoF
> incl. elevation), but there is **no mature, widely-adopted single standard for global volumetric
> sub-room addressing** today. Treat this as an open design point, not a settled choice.

### 3.2 Data formats — archival vs streaming vs incremental-update
The hardest cross-cutting truth: **streaming and archival pull in opposite directions, and
*incremental update* is the weakest property across nearly every format.** Most cloud-optimized
formats are byte-laid-out for read performance, so updating a region effectively means rewriting
the file.

**Point clouds**
| Format | Role | Streaming | Incremental update |
|---|---|---|---|
| **LAS / LAZ** (ASPRS) | Archival baseline (LAZ = lossless compressed) | No (full download) | Rewrite |
| **E57** (ASTM E2807) | **Archival** — points + imagery + scanner metadata, vendor-neutral | No | Rewrite |
| **COPC** (Hobu, 2021) | Single LAZ-1.4 file, octree in VLR, **HTTP range** | **Yes** (range requests, <5% index overhead) | Effectively rewrite (global byte layout) |
| **EPT** (Entwine) | Many-file octree, **lossless, additive** | **Yes** | **More update-friendly** (multi-file; append/replace nodes) |
| **Potree 2.0** | Web viewer octree (3 files) | **Yes** (WebGL LOD) | Rewrite |
| **3D Tiles `.pnts`** | *Deprecated* in 1.1 → use glTF `POINTS` | — | — |

**Meshes / scenes**
| Format | Role | Streaming + LOD | Incremental update |
|---|---|---|---|
| **glTF / GLB** (Khronos) | Runtime delivery (*not* authoring/archival) | payload only | n/a |
| **3D Tiles 1.1** (OGC, Cesium) | **Streaming tile tree** — bounding volume + geometric error + REPLACE/ADD refinement + implicit tiling + metadata | **Yes (the standard for this)** | Node/tile replace + append possible; refining LOD may force regenerating descendants |
| **I3S / SLPK** (OGC, Esri) | Competing node-tree streaming standard | **Yes** | Node-based, similar story |
| **OpenUSD / USDZ** | **Authoring + non-destructive layered editing** | no (not a tile format) | **Best incremental model** (composition arcs, sublayers, time-sampled attrs) |
| **CityGML 3.0** (OGC) | Semantic city model (tech-neutral), has a **versioning concept** | via I3S/3D Tiles | Conceptual versioning support |

**Raster / vector tiles**
| Format | Role | Streaming | Update |
|---|---|---|---|
| **COG** (OGC, 2023) | Cloud-optimized GeoTIFF, internal tiles + overviews | **HTTP range** | Rewrite |
| **PMTiles** | Single-file **serverless** Z/X/Y archive | **HTTP range, no tile server** | Rewrite |
| **MBTiles** | SQLite tile DB (needs a server) | server-mediated | row updates possible |
| **MVT** | Per-tile vector payload | — | — |

Sources: COPC <https://copc.io/> · E57 <http://www.libe57.org/> · EPT
<https://entwine.io/entwine-point-tile.html> · 3D Tiles 1.1 <https://docs.ogc.org/cs/22-025r4/22-025r4.html>
and <https://github.com/CesiumGS/3d-tiles> · glTF <https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html>
· I3S <https://docs.ogc.org/cs/17-014r9/17-014r9.html> · OpenUSD <https://openusd.org/release/intro.html>
· CityGML 3.0 <https://docs.ogc.org/is/20-010/20-010.html> · COG <https://docs.ogc.org/is/21-026/21-026.html>
· PMTiles <https://docs.protomaps.com/pmtiles/>

> **Update-friendliness ranking:** OpenUSD (layered overrides) > I3S / 3D Tiles (node/tile
> replace+append) > COPC / COG / PMTiles / Potree (effectively full rewrite). A realistic system
> *authors/edits* in something USD-like and *streams* in 3D Tiles / I3S.

### 3.3 Time / versioning — no turnkey 4D format
No mainstream 3D format has a first-class temporal model. 3D Tiles handles time via per-tile/
per-feature **metadata timestamps** + community conventions; OpenUSD via time-sampled attributes
(animation, not world history); CityGML 3.0 adds a versioning concept. **World-scale temporal
versioning is an application-layer problem**, not a format feature.

---

## 4. Open-Source Tools & Relevant Projects

A mature, well-segmented stack exists. No single product spans capture→store→serve — you compose.

**Processing / conversion / viewing (mature)**
- **PDAL** — "GDAL for point clouds": JSON pipeline of readers/filters/writers; reproject,
  classify, denoise, convert; streaming mode. <https://pdal.io/>
- **Entwine / untwine** — build EPT/COPC indexes from anything PDAL-readable; lossless;
  untwine single-pass <5% overhead. <https://entwine.io/>
- **Potree** — dominant open WebGL point-cloud viewer (TU Wien octree). <https://github.com/potree/potree>
- **Open3D / PCL / laspy / CloudCompare** — processing, registration, I/O, desktop GUI.
  <https://www.open3d.org/> · <https://pointcloudlibrary.github.io/> · <https://laspy.readthedocs.io/>
- **py3dtiles** — open LAS/XYZ → 3D Tiles converter. <https://gitlab.com/py3dtiles/py3dtiles>
- **CesiumJS / deck.gl (`Tile3DLayer`, loaders.gl) / MapLibre GL** — open rendering & basemap
  layer that consume 3D Tiles / I3S / LAS / point clouds. <https://deck.gl/docs/api-reference/geo-layers/tile-3d-layer>

**Storage / query engines — match the DB class to the job**
- **Bulk point storage in an RDBMS:** **PostGIS + pgPointcloud** (points grouped into `PcPatch`,
  GiST indexing) — good for moderate volumes/mixed workloads, *not* planetary serving.
  <https://pgpointcloud.github.io/pointcloud/>
- **Analytics at scale:** **DuckDB + spatial** (in-process, GeoParquet-native), **BigQuery GIS**
  (S2-based GEOGRAPHY), **ClickHouse** (H3/S2/geohash). <https://duckdb.org/2023/04/28/spatial>
- **Distributed/cluster:** **Apache Sedona** (Spark/Flink), **GeoMesa / GeoWave**
  (Accumulo/HBase/Cassandra) — powerful but operationally heavy.
- **Spatiotemporal:** **MobilityDB** (PostGIS moving-object types `tgeompoint`/`tgeogpoint`) —
  strongest open fit for trajectories over time; **TimescaleDB** for time-series partitioning.
  <https://github.com/MobilityDB/MobilityDB>
- **Cloud-native file serving (often the real "database" for a web map):** object storage +
  **PMTiles / COG / COPC / GeoParquet**, read via HTTP range requests — *no tile server*.
- **Versioned geodata:** **Kart** (Git-style VCS for vector/tabular/raster/point-cloud data;
  GeoGig is effectively unmaintained). <https://kartproject.org/>

> **On TigerBeetle (you raised it as an example):** it is a purpose-built double-entry
> *financial-accounting OLTP* database with a fixed accounts/transfers schema (~1M txn/s). It is
> **not a spatial or bulk store and should not anchor the design.** At most it could serve a
> narrow ancillary role as a high-throughput ledger for metadata/usage/billing events — and even
> that is optional. <https://docs.tigerbeetle.com/>

**Typical pipeline:** capture (LiDAR/photogrammetry) → **process** (PDAL/Open3D) → **index**
(untwine/Entwine → COPC/EPT; or py3dtiles/Cesium ion → 3D Tiles) → **store** (object storage for
tiles/COPC; PostGIS/warehouse for queryable subsets; MobilityDB for time-varying) → **serve**
(static range reads, or tile/feature server) → **render** (Potree / CesiumJS / deck.gl+MapLibre).

---

## 5. Major Unresolved Technical Challenges

Ordered roughly from "hard engineering" to "open research":

1. **Time-versioning of dense 3D data — OPEN.** 2D vector versioning is solved (OSM, SQL:2011,
   Kart). There is **no standard for versioning point clouds / meshes / NeRFs / splats** over
   time. <https://en.wikipedia.org/wiki/SQL:2011>
2. **Multi-contributor merge / conflict resolution for captures — OPEN.** OSM's optimistic
   locking (HTTP 409 on stale version) works for discrete vector elements; merging overlapping
   *dense scans* of the same place into one coherent model is not standardized.
   <https://wiki.openstreetmap.org/wiki/API_v0.6>
3. **Cross-capture 3D change detection without drift — OPEN.** ICP/GICP registration is solved
   for well-overlapping clouds, but odometry accumulates drift; city-scale change detection is a
   per-paper research problem. <https://arxiv.org/pdf/2103.14314>
4. **Freshness / confidence scoring — mostly OPEN.** No established standard; the most concrete
   work is recent autonomous-driving HD-map research (CleanMAP, RTMap). <https://arxiv.org/html/2504.10738v1>
5. **Storage & streaming/LOD economics for (4D) radiance fields at world scale — OPEN frontier.**
   3DGS is ~1 GB *per scene*; world-scale needs block-partitioning + LOD that is still being
   researched. <https://arxiv.org/pdf/2411.00771>
6. **Incremental update of cloud-optimized formats — hard engineering.** COPC/COG/PMTiles/Potree
   require effective full rewrites to change a region (see §3.2).
7. **World-scale localization (VPS) — difficult but deployed, proprietary.** Live at Google/
   Niantic; accuracy scene-dependent; internals not public. <https://developers.google.com/ar/develop/geospatial>
8. **Indoor semantic reconstruction — partially solved.** RoomPlan/ScanNet/Matterport3D give
   parametric layouts + semantics, but robust general indoor understanding is not solved.
   <https://machinelearning.apple.com/research/roomplan>
9. **Crowdsourced data-quality governance & incentives — OPEN.** OSM research shows completeness/
   accuracy vary by region/socioeconomics, with recurring contributor-quality threats.
   <https://www.researchgate.net/publication/335758476_Understanding_Threats_to_Crowdsourced_Geographic_Data_Quality_Through_a_Study_of_OpenStreetMap_Contributor_Bans>

---

## 6. Plausible Architecture Families (stated neutrally)

These are *families of approaches* surfaced by the survey, not a recommendation. Most real
systems blend several.

**A. Tiled-mesh streaming map (the "Google/Cesium" family).**
Capture → photogrammetry/mesh → bake into a 3D-Tiles/I3S LOD tree on object storage → serve via
range requests → render in CesiumJS/deck.gl. *Strength:* proven at planet scale, open standard.
*Weakness:* baking is heavy; time/versioning and incremental update are bolt-ons.

**B. Cloud-optimized point-cloud lake (the "PDAL/COPC/Entwine" family).**
Keep authoritative captures as COPC/EPT in object storage, indexed by a spatial cell (S2/H3) +
timestamp in a metadata catalog (PostGIS / DuckDB / a warehouse). Serve raw geometry by range
request; derive meshes/tiles on demand. *Strength:* lossless, archival, append-friendly.
*Weakness:* you build retrieval/LOD orchestration yourself.

**C. Localization-first / implicit map (the "VPS / LGM" family).**
Store a feature map (or neural weights) keyed to place; localize devices, then attach/retrieve
content. *Strength:* solves the "where am I, precisely" problem AR needs. *Weakness:* geometry
is often implicit; proprietary; freshness/versioning unclear.

**D. Semantic twin graph (the "Azure DT / iTwin / CityGML" family).**
Model places as versioned entities/relationships (DTDL/GERS-style stable IDs) with geometry
attached as assets; iTwin-style append-only Changeset ledger for true history. *Strength:* best
*versioning* story; queryable semantics. *Weakness:* not built for dense planet-scale geometry.

**E. Neural-representation map (the "NeRF / 3DGS" frontier family).**
Represent places as radiance fields, block-partitioned with LOD; 4D variants for time.
*Strength:* photoreal, compact-ish geometry-free capture. *Weakness:* world-scale + streaming +
editing + versioning are all open research.

**The realistic synthesis** most evidence points toward is a *layered, distributed, versioned
spatial data platform* — **not one giant file** — that combines: a standard global cell index
(S2/H3) + time as first-class keys (family B's catalog) → authoritative captures in
cloud-optimized formats → derived streaming tiles (family A) → an entity/version ledger for
history and merges (family D) → optional neural representations where they win (family E). Each
layer already has commodity pieces; the *connective tissue* (versioning, merge, freshness) is
where the real work is.

---

## 7. Best Next Readings (repos, standards, papers)

**Indexing**
- S2 cell hierarchy — <https://s2geometry.io/devguide/s2cell_hierarchy.html>
- H3 core library — <https://h3geo.org/docs/core-library/overview/>
- OGC DGGS Abstract Spec (Topic 21) — <https://docs.ogc.org/as/20-040r3/20-040r3.html>
- OGC GeoPose 1.0 — <https://docs.ogc.org/is/21-056r11/21-056r11.html>

**Formats / standards**
- OGC 3D Tiles 1.1 — <https://docs.ogc.org/cs/22-025r4/22-025r4.html> · spec repo
  <https://github.com/CesiumGS/3d-tiles>
- OGC I3S 1.3 — <https://docs.ogc.org/cs/17-014r9/17-014r9.html> · <https://github.com/Esri/i3s-spec>
- COPC — <https://copc.io/> · EPT — <https://entwine.io/entwine-point-tile.html>
- E57 — <http://www.libe57.org/> · glTF 2.0 — <https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html>
- OpenUSD — <https://openusd.org/release/intro.html> · CityGML 3.0 — <https://docs.ogc.org/is/20-010/20-010.html>
- COG — <https://docs.ogc.org/is/21-026/21-026.html> · PMTiles — <https://docs.protomaps.com/pmtiles/>

**Tools**
- PDAL — <https://pdal.io/> · Potree — <https://github.com/potree/potree> · py3dtiles —
  <https://gitlab.com/py3dtiles/py3dtiles>
- PostGIS/pgPointcloud — <https://pgpointcloud.github.io/pointcloud/> · MobilityDB —
  <https://github.com/MobilityDB/MobilityDB> · Kart — <https://kartproject.org/>

**Systems**
- Google Photorealistic 3D Tiles — <https://developers.google.com/maps/documentation/tile/3d-tiles-overview>
- Google ARCore Geospatial / VPS — <https://developers.google.com/ar/develop/geospatial>
- Niantic LGM — <https://nianticlabs.com/news/largegeospatialmodel>
- Overture GERS — <https://docs.overturemaps.org/gers/> · OSM full-history —
  <https://wiki.openstreetmap.org/wiki/Planet.osm/full>
- Bentley iTwin — <https://www.itwinjs.org/learning/imodelhub/>

**Papers (capture/representation frontier)**
- 3D Gaussian Splatting — <https://arxiv.org/abs/2308.04079>
- Block-NeRF — <https://arxiv.org/abs/2202.05263> · <https://waymo.com/research/block-nerf/>
- 4D Gaussian Splatting — <https://arxiv.org/pdf/2310.08528> · CityGaussianV2 —
  <https://arxiv.org/pdf/2411.00771>
- City-scale 3D change detection — <https://arxiv.org/pdf/2103.14314>
- Apple RoomPlan — <https://machinelearning.apple.com/research/roomplan>

---

## 8. Final Verdict — Commodity vs Tractable vs Open Research

| Capability | Verdict |
|---|---|
| Global Earth partitioning / addressable cells (S2, H3, quadkey, Plus Codes) | **Commodity** — use it, don't invent it. Your "cube grid" intuition ≈ S2. |
| Storing & streaming massive 3D geometry with LOD (3D Tiles, I3S, COPC, EPT) | **Commodity / solved engineering** |
| Consumer LiDAR/imagery capture & export (ARKit, Polycam, Scaniverse, RoomPlan) | **Commodity / deployed** |
| Cloud-native serving via object storage + range requests (PMTiles/COG/COPC) | **Commodity** |
| Spatial storage/query engines (PostGIS, DuckDB, BigQuery GIS, MobilityDB) | **Commodity** |
| Geometry/appearance/semantics as separate layers | **Tractable** — this is how mature systems already work; no unifying world-scale standard |
| 2D vector versioning + conflict resolution (OSM, Kart, SQL:2011) | **Commodity / solved** |
| World-scale visual localization (VPS) | **Difficult but deployed** (Google/Niantic; proprietary, scene-dependent) |
| Indoor semantic reconstruction | **Difficult / partially solved** |
| Sub-room / volumetric global addressing standard | **Open** — GeoPose is closest; no settled standard |
| Incremental update of cloud-optimized 3D formats | **Difficult engineering** (mostly full-rewrite today) |
| Versioning + merging **dense 3D / temporal** captures | **Open research — no standard** |
| Cross-capture 3D change detection without drift at scale | **Open research** |
| Freshness / confidence scoring | **Open research — no standard** |
| World-scale streaming/LOD for (4D) radiance fields | **Open research frontier** |
| Crowdsourced 3D data-quality governance & incentives | **Open** |

**In one line:** the storage, indexing, formats, capture, and serving are commodity or solved
engineering you can assemble today; the *defining, differentiated* parts of your idea —
**time-versioning of dense 3D, merging uneven multi-contributor captures, and freshness/quality
governance** — are exactly the parts that are still open research and very hard systems work.
That is where a real contribution would live.

---

### Sourcing & confidence notes
- Indexing, format, and standards claims are drawn from primary/standards sources (OGC, Khronos,
  ASPRS, s2geometry.io, h3geo.org, official project docs) and are high-confidence.
- **Flagged as proprietary / unverified:** Google VPS/Streetscape internals and 3D-Tiles update
  cadence; Niantic LGM scale figures and "shipped product" status; Cesium ion tiling internals;
  NVIDIA Earth-2 performance multipliers; Matterport/NavVis/Hexagon accuracy specs; the
  Scaniverse/SPZ "~90% reduction" and Niantic ownership (corroborated via search, not a primary
  doc). Treat marketing-scale numbers as indicative, not guaranteed.
- "Incremental update is a full rewrite" for COPC/COG/PMTiles/Potree is inferred from documented
  byte-layout structure rather than an explicit spec prohibition — moderate confidence.
- The "no mature 3D/indoor global addressing standard" and "no world-scale time-aware 3D map
  exists" conclusions are syntheses across sparse sources — treat as genuine open gaps, the most
  important and least certain findings, rather than settled facts.
