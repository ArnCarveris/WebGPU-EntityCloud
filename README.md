# Entity Cloud // WebGPU

Volumetric clouds and a weather system in WebGPU. Storm cells grow, drift with the wind, rain or snow,
flash with lightning and decay. Their precipitation shafts hang below the cloud base and stay visible
from far away: slanted by the wind, streaky, evaporating as virga, and lit from behind when the sun
gets under the storm.

Everything is in one static file, `index.html`: the engine classes, the WGSL, and the scenario as
data. Open it in a WebGPU browser (Chrome/Edge 113+, or Brave with WebGPU enabled), straight from
disk or served over HTTP:

```bash
python -m http.server 8769
```

Drop another scenario `.json` on the page to load it.

## Cloud types

The scenario includes the cloud types described in
[Climavision's cloud guide](https://climavision.com/blog/the-mysteries-of-clouds-types-formation-and-weather-predictions/):

| Cloud | Level | How it is made | Shown in |
|---|---|---|---|
| **Cumulus** | low | the convective layer: Perlin-Worley puffs with flat bases, cut by the weather map's coverage | fair cumulus, showers |
| **Stratus** | low | `sheet` genus layer: flat and featureless, drizzles where it is thick | stratus deck, overcast rain |
| **Altocumulus** | mid | `cellular` genus layer: separate Worley puffs, stretched across the wind into rows | mackerel sky, fair cumulus |
| **Altostratus** | mid | thin `sheet` genus layer: a grey veil that the sun shows through | warm front, thunderstorm |
| **Nimbostratus** | low to mid | thick, dark `sheet` genus layer with steady rain or snow | overcast rain, warm front |
| **Cirrus** | high | streaky sheet above the slab, stretched along the wind | clear, warm front |
| **Cumulonimbus** | all levels | `storm` cells: towers that narrow upwards, with an anvil spreading downwind | showers, thunderstorm |
| **Shelf cloud** | low | analytic (`shelfLineDensity`): stacked laminar tiers along a `squall` line's bowed gust front. Each tier is lowest at its rounded lip, has a flat dark underside and a sunlit top sloping up into the storm base. Single storms can also get a small one with `shelf` | Shelf cloud view (at dusk, light under the shelf) |
| **Mothership (supercell)** | all levels | analytic (`mothershipDensity`): a slowly spinning helix of plates around the `supercell` updraft. The plates have rounded lips, sunlit tops and dark undersides, and a wall cloud hangs under the stack. Inside the stack it replaces the noise cloud; the tower and anvil continue above it | Mothership view |
| **Green clouds** | — | not a cloud type: `green` tints the light under a heavy storm core | the hero supercell |

## How it works

Each frame runs these passes. All distances are metres and heights are above sea level.

1. **Weather maps** (`WGSL_WEATHER`, `WeatherPass`): a 512² map, 128 km across, that follows the
   camera and snaps to its texels. Each texel holds four values:
   - cloud coverage
   - cloud top
   - precipitation
   - virga (how much of the precipitation evaporates before it reaches the ground)

   The layer's coverage is large-scale noise advected by the wind. Every live storm cell then adds a
   noisy disc of coverage and a domed top. It also adds a precipitation core, which sits on the
   forward flank by default. The same pass writes three more maps:
   - **anvil + shelf**: anvil coverage and altitude, spread downwind of tall cells; shelf coverage and
     the wedge position along the gust front
   - **layers**: the coverage of each cloud genus, which drift with the wind
   - **style**: supercell plates, green tint, and how far the wall cloud lowers

   From these maps an **occupancy grid** (`WGSL_OCCUPANCY`) marks where cloud can exist: 256² columns of 500 m,
   each with 32 height cells (bits of one `u32`) from the lowest cloud to the highest top of the frame. A cell is set
   when it reaches into the convective layer below the column's own highest top (lowered under wall clouds), an
   anvil band, shelf height or a genus layer's band, taking the largest values over every weather texel the march
   can blend in that column.
2. **Cloud shadow map** (`WGSL_SHADOW`): 384² texels.
   - Sun transmittance is marched up through the cloud slab from the cloud base plane.
   - Sky occlusion is integrated over the cloud column above each texel (stored as the column's transmittance;
     the clouds' sky light and the lightning both read the column's optical depth from it).
   - The sun ray's optical depth below the top of the convective layer, so the clouds' lighting can take off the
     part of the ray below a point (`sunDepthAbove`).
   - Low cloud: the transmittance of the lowest 1.5 km of cloud above the base. Precipitation only falls where
     there is low cloud above the point it left the base (an anvil overhead does not count).

   Terrain, precipitation and haze sample it, so cloud shadows sweep across the ground and sunlit gaps
   cut crepuscular rays.
3. **Ground state** (`WGSL_GROUND`, `GroundPass`): 256² texels of snow cover and wetness. Snow
   builds up where precipitation reaches the ground below freezing, following the same slanted shafts (lapse rate 6.5 °C/km). Rain wets
   the ground. Snow melts and the ground dries afterwards.
4. **Scene** (`WGSL_SCENE`): the sky, then the terrain.
   - The terrain mesh is a geometric grid: linear over the heightfield, then rings growing out to
     260 km.
   - Ground shading: section-line farm fields, centre-pivot circles, towns, forests, rivers and lakes,
     and mountains with rock above the tree line.
   - It is lit with cloud shadows, snow, wet darkening and reflections, and lightning.
   - Then the **structures** (`vsStruct` / `fsStruct`): houses, a bus shelter, bridges, roads and the river's water in a
     village, and a town's central bus station, as flat-shaded triangles built on the CPU. They stand on `Heightfield.surface`, the terrain exactly as
     drawn (half-float heights, filtered at the mesh vertices, flat triangles between them), so they neither float
     nor sink. They get cloud shadows, wetness, snow on what faces up, and glass, metal and wet surfaces reflect the
     sky. See **Rain shelter** below.
5. **Froxel lighting** (`WGSL_FROXEL`, `FroxelPass`): a 160 × 96 × 128 volume of frustum-aligned cells (froxels),
   with depth slices spaced exponentially from 40 m out to the march distance. Each froxel stores the light at its
   centre:
   - sun transmittance: the cloud shadow map, times the shadow cast by the rain and snow shafts themselves (six
     steps toward the sun through the shaft envelope, without the streaks), so heavy shafts darken what lies behind
     them and the haze in their lee
   - sky light left under the cloud column
   - lightning

   Light changes slowly across space, so a coarse grid holds it. The march reads it with one trilinear fetch per
   step below the cloud base, in place of the shadow map lookups and the loop over flashes. The densities stay
   per step, so the streaks keep their detail. `F` switches it off, and the march then evaluates the light per step
   (without the shafts' own shadow).
6. **Volumetrics** (`WGSL_TILES`, `WGSL_MARCH`, `CloudPass`): first a **cloud tile pre-pass**: one ray through the
   centre of each 4 × 4 block of volumetric pixels takes one shape sample per distance bin (64 bins, square-root
   spaced out to the march distance), with every noise cloud's coverage padded by 0.15 so its clouds come out a little
   larger, and records the bins that hold cloud. Then a compute ray march at reduced resolution. Steps are
   spaced quadratically (dense near the camera), with a jittered start. Each step integrates three
   media, energy-conserving:
   - **Clouds** (`cloudSample`): the convective layer and storms, shelf clouds, and up to four genus
     layers, summed at each sample. Each genus has its own ambient-light scale, so nimbostratus is dark
     and altostratus is pale.
     - **Shape**: a baked 128³ Perlin-Worley shape volume, cut by coverage.
     - **Detail**: a 32³ Worley volume erodes edges: wispy at the base, billowy at the top.
     - **Vertical profile**: flat bases, rounded tops, an anvil band, and wind shear that leans the
       towers.
     - **Light**: a short march toward the sun, then the rest of the sun ray from the shadow map, so a low sun
       does not light the underside of a deck it would have to cross for tens of kilometres. Three
       multiple-scattering octaves (Wrenninge).
     - **Sky light**: near a cloud surface the view ray reached through open air, the sky lights the cloud from the
       side (less under tall towers). Deeper in, and when the camera is inside the cloud or the rain, only the light
       that diffused down through the cloud above is left (`cloudAbove`): its transmittance is about
       1 / (1 + 0.11 τ) for the optical depth τ above, so the middle of a storm is dim and it brightens toward the
       tops. That light flows downward, so it is brighter looking up than down (`diffuseLook`). Rain under a storm
       is lit the same way.
     - **Lightning** (`flashLit`): a flash fires inside the cloud and its light diffuses out through the storm
       column, 1 / (1 + 0.11 τ) for the cloud between the flash and the lit point. The cloud around the flash
       glows, while the base, the rain shafts, the haze and the ground below get a dimmer, spread-out light.
   - **Rain and snow**: precipitation below the cloud base.
     - **Slant**: storms move with the wind aloft and the slower air below holds the falling precipitation
       back, so a shaft trails behind its cloud. Each sample reads the weather where its precipitation left the
       cloud base, downwind by `wind · drop · slant` (`precipSourceAt`): the foot of the shaft lies upwind of the
       cloud.
     - **Source**: the shafts hang from the local cloud base: the layer's base, or the bottom of a mothership's
       plate stack, which hangs lower (`precipBase`). The rate is cut where no low cloud hangs above the point the
       precipitation left the base, so a rain core on a storm's flank, carried further by the wind, does not fall
       from clear sky beside the cloud or from under an anvil (`rainCover`).
     - **Streaks**: falling curtains of stretched 3D noise, travelling with the clouds.
     - **Virga**: the shaft fades out above the ground. A downpour overwhelms it: virga fades from intensity 0.6
       and is gone by 1.4 (`virgaOf`).
     - **Heavy cores**: a storm cell's precipitation core is flat-topped, so its whole middle pours. Precipitation
       can pass 1 (up to 2) when the weather state's `power` is above 1, as in severe storms. From 0.6 up, the
       shaft gets denser than in proportion and its streaky curtains merge into a solid wall, and the HUD reports
       torrential rain under it.
     - **Rain or snow**: rain below the freezing level and snow above it, each with its own extinction
       and phase. Rain scatters strongly forward, so backlit shafts glow.
   - **Haze**: height fog that is shadowed by the cloud map, and tends to the horizon sky colour far
     away. Analytic haze covers the terrain beyond the march.
   - **Cirrus**: a streaky sheet above the slab.

   A **temporal resolve** reprojects the history using the transmittance-weighted depth, clips it to
   the variance of the pixels marched this frame nearby, and blends.
7. **Final** (`WGSL_FINAL`):
   - Composite and ACES tonemap.
   - Near-field rain streaks and snowflakes in a box that wraps around the camera. Their amount comes
     from a one-texel GPU readback of the weather map and the shadow map's low cloud above the camera. How the rain
     looks follows the state's drop size (see **Rain variants**), and rain runs off the roof edges near the camera.
   - Lightning bolts.
   - A weather radar inset, showing the precipitation that leaves the cloud base. Rain shows in green, yellow and red, snow in blues, and virga aloft in
     grey-blue.

Under cloud, the ground's sun light is the shadow map's transmittance. Points inside the cloud slab get progressively
more sun (the map's march from the slab's base overstates the cloud toward the sun there), measured from the real
cloud base: the lowest genus layer or the convective base, not the march's padded lower bound.

The sky and sun colours come from the sun elevation, using Kasten-Young air mass through Rayleigh
and Mie extinction. They turn greyer and darker with cloud coverage.

## Performance and LOD

The HUD's `gpu` line shows the GPU time of each pass. It uses timestamp queries (`GpuProfiler`) when the adapter
offers them.

- **Interleaved march** (`interleave` per quality preset): low, medium and high march one pixel of each 2 × 2 block
  per frame, taking turns over four frames (diagonal first); ultra marches every pixel. A setting of 2 marches a
  checkerboard. The resolve rebuilds the other pixels by reprojecting the history, clamped to the pixels marched
  this frame within two pixels (4 to 9 of them), and blends a freshly marched pixel in with a weight of 0.25. One in
  four instead of one in two took the march from 1.5 to 0.9 ms at medium and from 4.3 to 2.3 ms at high. While
  turning, the clouds come out a little softer than with the checkerboard, without trails.
- **Cloud blur** (low and medium quality): the composite smooths the half-resolution volumetrics with a 3 × 3 tent of
  bilinear taps (radius 1 volumetric texel at low, 0.7 at medium). A tap counts less where its transmittance differs from
  the centre pixel's, so the march noise inside clouds smooths out while cloud edges and the terrain outline stay sharp.
  It runs after the temporal resolve, so the history never blurs further. About 0.1 ms.
- **Distance LOD**: `cloudSample(p, w, lod, feat)` has three levels:
  - lod 0, within the preset's `detail` distance: detail-noise erosion, grooves, striations and scud
  - lod 1, up to 2.5 × that distance: no erosion
  - lod 2, beyond that, and always for light marches and the shadow map: shape only

  Light marches take fewer, longer steps with distance.
- **Step-count LOD**: short segments, such as rays looking down at nearby ground, get fewer march steps (at least
  24).
- **Empty space**: march steps in cells the occupancy grid marks empty skip the cloud noise entirely; only the
  motherships and shelf lines the ray passes near are still evaluated. Most empty air lies above a column's own top,
  since the slab reaches up to the tallest storm anywhere. Elsewhere, after two cloud-free samples, clouds are
  evaluated only every other step until one is hit. Nothing is sampled below the lowest cloud or above the highest
  cloud top of this frame. The grid costs about 0.2 ms and saves 0.6–1.3 ms of march. A bound on the shape noise in
  the grid was tried too: it skipped little more and cost as much as it saved.
- **Cloud tiles** (`G` switches them off): the march evaluates clouds only in the distance bins the tile pre-pass found
  cloud in, over the pixel's tile and its eight neighbours, widened by one bin each way. The pre-pass costs 0.4 ms at
  medium and 0.7 ms at high and takes 1.0–2.2 ms off the march. It ignores the scene depth (a tile can straddle the
  horizon) and evaluates every mothership and shelf line. Two samples per bin cost twice as much and skipped no more.
- **No volumetrics** render mode skips the froxel, tile, march and resolve passes.
- **Culling**:
  - Each ray tests the bounding boxes of the motherships and shelf lines once, and only evaluates those it can meet.
  - Shelf lines reject points against the line's chord before the curve solve.
  - The anvil and style maps are only read at heights and places where those clouds can exist.
  - The weather map skips storm cells beyond their reach, and skips each cell's noise where it cannot matter.
- **Froxel lighting**: the precipitation and haze light is computed once per froxel (about 2 million, about 0.1 ms)
  instead of at every march step.
- **Time slicing**: the cloud shadow map refreshes a quarter of its rows per frame. It refreshes fully when the map
  moves, the sun moves, or the weather changes.

On the test machine (Brave, about 1925 × 925 pixels), GPU time per frame dropped from 11–13 ms to 4–6 ms at medium,
and from 25–28 ms to 10–12 ms at high.

## Controls

| Key | Action |
|---|---|
| drag / WASD / Space, C | look / move / up, down |
| Shift, Alt, wheel | ×5, ×0.2, speed |
| right click (or Ctrl + click) | grow a storm cell where the cursor meets the ground |
| N | rain variants: drizzle, light rain, medium rain, downpour (see **Rain variants**) |
| 1–9 | weather states: clear, fair cumulus, mackerel sky, warm front, stratus deck, showers, thunderstorm, snow squalls, overcast rain (they blend over `transition` seconds). A 10th, severe storms, is reached by auto-cycle and by the shelf and mothership views |
| 0 | auto-cycle the weather states |
| K | lightning from the nearest raining cell |
| , . | weather time scale (×1 … ×160) |
| T, arrows | next lighting preset; move the sun (azimuth, elevation) |
| R | radar inset |
| F | froxel lighting on / off |
| G | cloud tile pre-pass on / off |
| M | render mode: shaded, no volumetrics (skips the volumetric passes), clouds only, precipitation only |
| Q | quality: low / medium / high / ultra (volumetric resolution, steps, light steps, interleave, detail distance, cloud blur) |
| V, L, P, H | next view, labels, pause weather, help |
| X | walk (from the ground below the camera) / fly |
| B | go to the next bus of the line (each press another): outside its front door while it stands at a stop, else aboard in the aisle |
| E | aboard: sit in the seat you look at, or stand up |
| hold Z | the buses run ten times as fast |

On foot, WASD walks (Shift runs, Alt creeps) and dragging looks around; Space and C do nothing.

### Rain variants

Four weather states give steady rain from a stratus or nimbostratus deck at four strengths, in light winds (3–7 m/s),
so the rain falls at no more than about 40° and a roof keeps it off what stands under it. `N` steps through them;
the scenario's bus station views start in medium rain, a downpour and light rain.

| State | Deck | `rain` | `drops` | Near-field intensity | Looks like |
|---|---|---|---|---|---|
| drizzle | stratus | 0.8 | 0 | about 0.1 | a slow mist of fine droplets that drifts with the wind (2.5 m/s fall) |
| light rain | nimbostratus + stratus | 0.38 | 0.35 | about 0.2 | sparse, thin, short streaks |
| medium rain | nimbostratus + stratus | 1.0 | 0.6 | about 0.55 | steady streaks; water drips off roof edges |
| downpour | thick nimbostratus | 2.6 | 1 | about 1.4 | dense, long, fast streaks (10 m/s), up to three times the drops, a grey wall of rain shafts, sheets of water off the roofs |

- **Rate**: `rain` scales the genus layers' precipitation in the weather map. Above 1 the excess adds on top, so a
  thick deck reaches intensity 1.4 like a severe storm's core: the shafts merge into a solid wall and visibility drops.
- **Drop size** (`drops`, in the frame as `rain.x`): the fall speed is 2.5 + 7.5 · drops^0.7 m/s. The near-field drops
  fall at that speed; fine drops come in a smaller, denser box (40 m instead of 70 m across), as faint thin specks; big
  drops are wider, brighter streaks, and past intensity 0.6 the particle count grows to three times `particles`. The
  rain shadows use the same speed, so drizzle blows further in under a roof than a downpour does.
- **Roof runoff**: roofs register their edges with `Structures.drip` (the station's canopy and terminal, the village
  bus shelter, every house's eaves). Each frame the 8 nearest within 150 m go into the frame (`dripEdges`), and
  40 000 extra particles fall from them (`drip` in `vsPrecip`): each picks an edge (nearer and longer ones get more),
  a side, and a spout within 30 m of the camera's place along it, and falls freely to the ground, leaning downwind.
  They are thinned to at most 100 per metre of edge and grow with the rain rate.
- The HUD names the rain by its rate and drop size: drizzle, light, moderate, heavy, torrential, downpour.

### Rain shelter

Structures stand in the shaders as up to 128 boxes (`blockers` in the frame uniform), each turned by its yaw and
sheared along its length by a slope, so bridge decks follow their ramps and arch. A house is two boxes (walls; roof
with its eaves), the bus shelter four (roof, back and side panels), a deck one per 8 m piece, fitted inside the
curved deck, and the bus station about 31 (canopy, fascia, clerestory, pillars, kiosk, buses, terminal, town blocks). `WGSL_SHELTER` tests rays against them:

- **Rain shadow**: a point is dry where the path its drops came along, straight back up against their fall and slanted
  by the wind (0.8 of the wind, rain at the state's fall speed, snow at 1.35 m/s, as the near-field particles), enters a box. So the
  ground under a roof or a deck stays dry and the dry patch shifts downwind; a back wall keeps the rain off the bench
  when the wind blows from behind it; snow blows in further.
  - Terrain and structures: no wetness and no settled snow there.
  - Near-field drops and flakes: none inside the dry volume (`vsPrecip` tests each particle along its own fall).
  - Rain and snow shafts: rays through the boxes' bounds take the shaft density out wherever it is sheltered.
  - The HUD says `sheltered` when the camera is, and `in the bus` when it is in the bus's cabin.
- **Sun shadows**: the ray toward the sun, so houses shade the street and each other, and decks the river bank.
- **Sky occlusion**: under roofs and decks (a box's `ao`), less sky light reaches the ground.

A ray starting inside a box does not count, so surfaces do not shade themselves. Only boxes within 2.5 km of the camera
go into the frame (the village and the bus station lie 9 km apart). Each frame a 32 × 32 grid over the
boxes' bounds lists, per cell, the boxes that can shade it: a capsule from each box toward where its sun shadow and its
rain shadow fall, as far as a ray from the ground under it to its top runs sideways. A point tests only its cell's
boxes, which keeps the structures at about 0.2 ms of scene time at street level.

### Walking and the bus

`X` puts you on foot (`Walker`). You walk on the terrain as drawn and on everything `Structures` built: every box
(`Structures.box` records it as a solid, see `solid`), and the bridges' decks and parapets. Each frame the walker tests
the solids within 2 m, from a 16 m grid over them. A top within 0.5 m of your feet is a floor you step up onto. A box
that reaches higher is a wall that pushes you out, unless all of it is above your head. You cannot wade into open water,
but you can cross it on a bridge. On foot the near plane comes in from 1 m to 5 cm, so the seat backs in front of you are
not clipped.

A `bus` entity (`BusLine`) runs a fleet of buses (`Bus`) between the bus station and the village:

- **Fleet**: `fleet` buses, by default the station's `buses` (5), so every bus of the station runs the line and none is
  left parked at the bays. The first is the entity's `label` in its `livery`; the others are numbered after it
  (`Bus 7 #2` ...) in the station's liveries. They leave the station at even intervals: one trip round (both dwells
  included, about 21 minutes in the default scenario) over the fleet, about 4 minutes apart with 5 buses. At load the
  trip is timed with a lone bus, and each bus is run on to its place in the timetable, so they are spread along the
  route from the first frame and pass each other on the road. A bus never closes up to within 6 m of the one ahead:
  it brakes for it as for a stop and waits there if it must.

- **Route**: from the station's platform it leaves past the east end of the canopy and runs along the south side of the
  town blocks. A two-lane road with a dashed centre line, built by the entity, takes it to the village road's nearer end.
  It drives through the village, crossing the stone bridge on its deck, round a balloon loop past the far end, back
  through the village, and in through a gap in the forecourt railing to the platform again. The station leaves its
  drive-through lane to the bus line. The path is rounded at the corners with arcs and resampled every metre, so the
  bus's position is an O(1) lookup. The bus follows it by its axles: the heading runs from the rear axle to the front
  one, and their heights pitch it.
- **Driving**: the bus cruises at `speed` (default 22 m/s), slows to `village` (10 m/s) in the village and to 6 m/s at
  the station and round the loop, and takes bends at 1.3 m/s² sideways. It brakes at 1 m/s² ahead of slower stretches
  and stops, and pulls away at 1.1 m/s². It stops at the platform and by the village's shelter, on whichever pass has
  the shelter on its right, with the front door at the shelter. At each stop it waits `dwell` seconds with the doors
  open; they close 3 s before it leaves. A one-way trip takes about 11 minutes; hold `Z` to speed it up.
- **The bus**: 12 m long, built in its own frame (`buildBus`; `BUS` holds its sizes). It has a low floor, eight window
  bays a side with pillars between them, and two glazed doors on the right (front and middle) whose leaves slide apart
  outside the body. There are 41 forward-facing seats (raised over the wheel arches), a back bench, poles and rails, a
  driver's cab with the driver, ceiling lamps, head, tail and destination lights, a windscreen and a rear window. The
  scene pass draws each bus within 5 km with its own uniforms (`wgslBus`: one 256-byte slot per bus, bound at a dynamic
  offset). Its clip matrix is built in doubles about the camera, so an interior a metre away does not jitter at 12 km
  from the origin. Door leaves carry a flag and move in the vertex shader (`busVertex`).
- **Glass**: the windows are drawn in the final pass after the rain particles, without depth, so the march, the haze
  and the rain outside show through them (`fsGlass`). They have a faint tint and reflect the sky at grazing angles.
  While it rains they carry drops, from a hash per 2 cm cell; on the side windows the wind of the bus's speed draws the
  drops out backwards into streaks.
- **Inside**: the cabin is lit through its windows (`shadeCabin`): sky light, less below the window line; the sun only
  where its ray leaves the body through glass, a single box exit (`throughGlass`), so sunlight falls in window-shaped
  patches; and the ceiling lamps. A canopy over the bus still shades it.
- **Riding**: step in through an open door and you ride in the bus's frame. Your position is kept in bus coordinates,
  and your view turns with the bus. Look at a seat within reach and press `E` to sit; your eye is then the seat's. Get
  off through an open door. The bus's colliders (floor, walls with door openings, seats, wheel arches, driver's cab, and
  the door leaves while the doors are closed) are boxes in its frame. Outside the bus they are moved into the world
  each frame.

**The cabin is a rain shelter that moves**, and it costs O(1). The frame uniform carries the body of one bus, the one you
ride or else the nearest, as one box in its frame (`busInv`, `cabinLo`, `cabinHi`):

- Near-field drops, flakes and roof drips: one point-in-box test per particle (`inCabin`); none fall inside.
- Volumetric march: one ray-box test per ray (`cabinSpan`); the steps inside the cabin take no rain, snow or haze. On
  the test machine that made the march cheaper, not dearer, from the aisle in a downpour: 0.21 ms with the cabin
  against 0.55 ms without, with every other pass the same.
- Every bus's body is also a moving shader box (`dyn`): it casts a sun shadow, keeps the rain off what is in its lee, and
  occludes the sky under it. It leaves no dry patch on the wet road (`structureLight` skips moving boxes for wetness).

### Analytic storm structures

Shelf lines and motherships are shapes, not noise. Entities report them through a `features(out)` hook (`out.ms`,
`out.shelves`), and the frame uniform carries up to 4 motherships and 2 shelf lines (`ms`, `shelves`, `features`).
The march, the light march and the shadow map evaluate them like any other cloud, so they cast shadows and are lit
the same way. Their tops get more sky light than their undersides, which is what makes the plates and tiers read.

## Scenario (`<script id="scenario">`)

| Key | Contents |
|---|---|
| `terrain` | `size` (m), `resolution`, `base` height, `fieldSize` (m, farm sections), `pivots` (chance of a centre-pivot circle per section) |
| `render` | `quality` (0–3), `shapeScale` / `detailScale` (m per noise tile), `detailStrength`, `maxTop` (top of the cloud slab), `maxDistance`, `weatherSize` (m), `rainExtinction` / `snowExtinction` (1/m at full intensity), `slant` (s/m), `fallSpeed` (streak scroll, m/s), `timeScale`, `particles` |
| `clouds` | up to 4 cloud genus layers (below) |
| `weather` | `start`, `transition` (s), `cycle` { `enabled`, `hold` }, `states` { name: state } |
| `entities` | `{ type, id, label, ... }`, where `type` maps to a class in `ENTITY_TYPES` (below), applied in order |
| `lighting` | `start`, `presets` { name: { `azimuth`, `elevation`, `intensity`, `exposure` } } |
| `views` | `{ name, pos [x, y, z], look [x, y, z] }`, or `{ name, follow (entity id), offset [x, height above ground, z], lookOffset }` to frame a moving entity, or `{ name, follow (entity id), spot }` for a viewpoint the entity laid out (a village's; a bus's `seat` puts you in one); optional `lighting` (preset), `weather` (state) and `walk` (true: on foot from there) |

Positions are metres: `[x, z]` on the map, with x east, z south, and the map centred on 0.

### Cloud genus layer (`clouds`)

| Key | Meaning |
|---|---|
| `name` | the key that weather states use in `layers` |
| `kind` | `heap` (puffy), `sheet` (flat, featureless) or `cellular` (separate puffs) |
| `base`, `top` | m above sea level |
| `density` | extinction (1/m) |
| `mapScale` | size of the coverage patches (m) |
| `shapeScale` | noise tile (m) |
| `stretch` | along the wind; < 1 makes rolls across it |
| `erosion` | detail erosion of the edges |
| `ambient` | light scale, < 1 is darker |
| `precip` | drizzle at full coverage |

The weather map writes each genus's coverage into one channel of a layer map (`layerTex`).

### Weather state

| Key | Meaning |
|---|---|
| `coverage` | cumulus (convective layer) coverage, 0–1 |
| `base`, `top` | cumulus base and top (m) |
| `layers` | `{ genus: coverage }` for the `clouds` genera; blended between states like everything else |
| `density` | cloud extinction (1/m) |
| `cirrus` | cirrus sheet coverage |
| `wind` | `[x, z]` m/s. It advects the clouds, moves the cells, slants the shafts and blows the near-field rain and snow |
| `temperature` | °C at sea level. It sets the freezing level and whether precipitation falls as rain, sleet or snow |
| `haze` | aerosol amount |
| `drizzle` | precipitation from the layer itself, where it is thick |
| `storms` | multiplier on the spawners' rate |
| `power` | multiplier on storm precipitation; above 1, cores reach torrential intensity (up to 2) |
| `lightning` | multiplier on flash rates |
| `rain` | multiplier on the genus layers' precipitation (default 1); past 1 a thick deck pours, past intensity 1 |
| `drops` | drop size, 0 (drizzle) to 1 (downpour), default 0.8: fall speed, and how the near-field rain looks |

### Entity types

| Type | Kind | Parameters |
|---|---|---|
| `tilt` | terrain | `dir` (downhill), `drop` (m across the map) |
| `hills` | terrain | fractal noise: `amplitude`, `scale`, `octaves`, `seed`, `ridged` |
| `mountain` | terrain | `pos`, `radius`, `height`, `roughness` |
| `range` | terrain | ridged massif along `path`: `width`, `height`, `roughness`, `seed` |
| `river` | terrain | carves and paints water along `path`: `width`, `depth`, `bank` |
| `lake` | terrain | `pos`, `radius`, water `level` |
| `town` | terrain | `pos`, `radius`, `density` (street blocks) |
| `forest` | terrain | `pos`, `radius`, `density`, `seed` |
| `village` | structure | a village on a road, laid out from `seed` (below) |
| `busStation` | structure | a town's central bus station (below) |
| `bus` | vehicle | a bus line from a `busStation` (`from`) to a `village` (`to`), and the road between them: `speed`, `village` (m/s), `dwell` [station, village] (s), `livery` [r, g, b], `fleet` (default: the station's `buses`); viewpoint `seat` (the first bus). See **Walking and the bus** |
| `storm` | weather | storm cell, below |
| `supercell` | weather | a storm with a mothership plate stack: `stackRadius` (× radius), `stackDrop` / `stackHeight` (m below / above the cloud base), `plates`, `twist` (plates climbed per turn), `spin` (rad/s), `wallCloud` (m), `wallRadius` (× stack radius) |
| `squall` | weather | a line of `count` storms from `from` to `to`, carried by the wind (`drift`, `velocity`); it starts over after `travel` m. Also takes `radius`, `top`, `precip`, `lightning`, `green`. Its shelf cloud runs along the gust front, `gap` × radius ahead of the cells and bowed out by `bow` m: `shelf` (strength), `lip` (m above ground), `shelfDepth`, `tiers` |
| `spawner` | weather | spawns storm cells inside `area` [x0, z0, x1, z1]. `rate` (cells per weather hour, × the state's `storms`), `max`, `initial`, `seed`. `template` gives a `[min, max]` range per storm parameter (`radius`, `top`, `precip`, `core`, `grow`, `mature`, `decay`, `lightning`, `virga`, `drift`) |

A `village` has these parameters:

- `pos`, `radius` (m): the main road runs `radius` m each way through `pos`
- `river`: the id of a `river` entity. The road crosses it square on a stone bridge with parapets, which ramps up onto
  piers over the water; without one, the road runs at `road` degrees from +x
- `houses` (at most): along the main road and a side street that branches off on the longer bank, along the river.
  Each has walls, a plinth, a gable roof (along or across), a door, windows per storey, and maybe a chimney
- `footbridge` (m): a wooden footbridge with railings this far along the river, in the direction of its `path`
  (negative: the other way; 0: none), with paths to it
- `busStopSide` (1 or -1): which side of the road the bus shelter stands on, its back to the street's edge. By
  default the back faces west, where the rain comes from in most states
- `channel` (m, default 3): the village deepens the river's channel locally and draws the water itself at the river's
  true width, since the land-use map's cells (125 m) are wider than the river
- viewpoints for `views` (`{ follow: "village", spot }`): `bus stop` (under the shelter), `bridges`,
  `under the bridge`, `overview`

A `busStation` is a big rain shelter in the style of a 1970s concrete bus station: a long cantilevered canopy with a
deep fascia, a ribbed soffit and a glazed clerestory along its spine, on one row of square pillars. Under it an island
platform has benches, timetables, a kiosk and bay signs; buses stand nose-in at the bays on one side and a
drive-through lane runs along the other. A terminal building with a flat overhanging roof closes the west end, and town
blocks stand around the forecourt. The ground is levelled under it. Parameters:

- `pos`, `yaw` (degrees; the canopy runs along it), `length` and `width` (m of canopy, default 96 × 30: it overhangs
  the 15 m platform by 7.5 m each side, so wind-blown rain does not reach it)
- `bays` (default 20), `buses` (default 5: nose-in at random bays, one in the lane; when a `bus` line runs from the station they all run it instead), `seed`
- `level` (m, default 200): the terrain is flattened to its mean within this radius, blending out 300 m further
- viewpoints for `views` (`{ follow: "station", spot }`): `platform` (under the canopy, looking out over the bays),
  `forecourt` (out in the rain at the canopy's corner), `overview`
- with a `bus` line from it, the drive-through lane is the bus's and the forecourt railing has a gap for it

The canopy's soffit is 5.4 m up and its fascia hangs to 4.7 m. It is one shader box from the bottom of its fascia to the top of the upstand, so the platform stays dry and the
deep fascia keeps slanting rain off its edge; the pillars, the kiosk, the buses, the terminal and the town blocks are
boxes too. The scenario places it in Brightfield at `[-3600, 8400]`.

A `storm` cell has these parameters:

- `pos`, `radius`, `top` (m)
- `precip` (0–1)
- `core` (precipitation core as a fraction of the radius), and `coreOffset` [x, z] (m; default: the
  downwind flank)
- `life` [grow, mature, decay] in weather seconds
- `lightning` (flashes per minute while mature)
- `virga` (0 reaches the ground, 1 evaporates at the base)
- `drift` (× wind) and `velocity` [x, z]
- `pinned` (stays in place and mature), `hold` (only stays mature)
- `shelf` (0–1, shelf cloud on the gust front)
- `laminar` (0–1, mothership plates)
- `green` (0–1, green tint under the core)
- `wallCloud` (m the base lowers under the updraft)

The cell moves through its life like this:

- **Growth**: the cell grows, and its top rises.
- **Maturity**: precipitation starts.
- **Decay**: coverage fades, the anvil lingers, and precipitation turns to virga.

### Adding a new kind of entity

1. Subclass `Entity` and override the hooks it needs:
   - `stamp(field)`: shape `field.h` and paint land use with `field.paint(idx, channel, value)`, where
     the channel is 0 town, 1 forest or 2 water.
   - `build(structures)`: after every entity has stamped, add meshes (`box`, `sweep`, `quad`, or `house`,
     `busStop`, `bridge`, `road`, `water`) and shader boxes (`blocker`) to the `Structures`. Place them with
     `structures.ground(x, z)`, the terrain as drawn.
   - `spawn()`: register with the world. `world.add(def)` creates more entities at runtime.
   - `update(dt, wdt, t)`: per frame. `dt` is real seconds and `wdt` is weather seconds. Set `dead` to
     remove the entity.
   - `cell()`: return `{ x, z, radius, top, coverage, precip, seed, virga, core, offset }` to put a
     storm cell into the weather map, or `null`.
2. Register it in `ENTITY_TYPES` and add it to the scenario's `entities`.

## Layout (sections of the script in `index.html`)

```
config, math, noise                        constants, vectors / matrices, half floats, value noise, polylines
Heightfield                                CPU terrain + land use: authoring, sampling, packing, raycast
FRAME_LAYOUT, FrameBlock, WORLD_BINDINGS,   uniform block and bindings shared by JS and WGSL (the WGSL
BindingSet                                 struct and declarations are generated from the same lists)
WGSL_MATH / SKY / WEATHER_SAMPLE /         shared WGSL pieces, composed per pass
DENSITY / SHADOW_SAMPLE
WGSL_OCCUPANCY, WGSL_NOISE, WGSL_WEATHER,  compute: occupancy grid, noise volumes, weather + anvil maps, shadow map,
WGSL_SHADOW, WGSL_GROUND                   ground state
WGSL_FROXEL, WGSL_SKIP_SAMPLE,             compute: froxel lighting, occupancy / tile lookups, cloud tile pre-pass,
WGSL_TILES, WGSL_MARCH, WGSL_RESOLVE       volumetric march, temporal resolve
WGSL_SCENE, WGSL_FINAL                     render: sky / terrain; composite, precipitation particles, bolts, radar
WGSL_SHELTER                               structure boxes: rain shadows, sun shadows, sky occlusion; the bus cabin
Entity, ENTITY_TYPES                       terrain features, GroundFrame / Structures / Village / BusStation, BusLine (+ buildBus,
                                           roundPath, offsetLine), StormCell, Supercell,
                                           SquallLine, Spawner
WeatherSystem, CloudLayers, Sky,           data-driven weather (states blend every value, including genus
Lightning, World                           coverage), cloud genus layers, sky colours, lightning, world
NoiseVolumes, WeatherPass, GroundPass, FroxelPass, CloudPass, GpuProfiler, Renderer
Input, FlyCamera, collideWalker, Walker,    input, flying, walking and riding,
Hud, App, main                             HUD, app
```
