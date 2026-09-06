# Flight Simulator 2026

*Jakub's Stuffs*

A 3D flight simulator in **one HTML file**. Three.js is the only external
dependency (loaded from a CDN, with two fallbacks); everything else — terrain,
airports, aircraft, textures, instruments and sound — is generated at load
time. No asset files, no build step, no server.

**[index.html](index.html)** — open it, or host it anywhere static.

## Hosting

Because it is a single self-contained file named `index.html`, it works as-is:

- **GitHub Pages** — Settings → Pages → deploy from this branch, root folder.
- **Cloudflare Pages** — connect the repo, no build command, output directory `/`.
- **Locally** — double-click the file. It needs one network request on first
  load to fetch Three.js; everything after that is offline.

## The flight model

Six-degree-of-freedom rigid body. Aerodynamic forces come from angle of attack
and sideslip through a conventional derivative set, and moments are integrated
through Euler's equations with each type's real inertia tensor. The physics runs
on a fixed 120 Hz accumulator, so it stays real-time on a slow machine instead
of quietly running in slow motion.

Nothing is scripted. Stalls, spins, ground loops, wing strikes and Dutch roll
all fall out of the same equations. The numbers were tuned until the simulated
performance matched published figures:

| | Cessna 150 | A320-style | 737-800-style |
|---|---|---|---|
| Stall, clean | 45 kt *(book 48)* | 150 kt *(book 148)* | 154 kt *(book 152)* |
| Stall, full flap | 39 kt *(book 42)* | 111 kt *(book 108)* | 115 kt *(book 112)* |
| Climb, sea level | 670 fpm *(book 670)* | 5 900 fpm | 6 200 fpm |
| Climb at 29 500 ft | — | 2 240 fpm | 2 010 fpm |
| Ground roll | 331 m @ 58 kt | 1 171 m @ 157 kt | 1 380 m @ 170 kt |
| Cruise | 90 kt, 68 % power | M0.78, 88 % | M0.78, 92 % |

They fly differently because the derivatives differ, not because of a difficulty
setting:

- **Cessna 150** — light, twitchy in roll, and it drops a wing at the stall.
  Full power at low speed pulls left and needs right rudder. Fixed gear.
- **A320-style** — side-stick fly-by-wire. The stick commands load factor, not
  elevator; it auto-trims, coordinates its own turns, and will not let you
  stall or overbank. Lose hydraulics or AC power and it reverts to direct law,
  taking the protections with it.
- **737-800-style** — cables and a trim wheel. Heavier in pitch, no protections,
  and out-of-trim forces are yours to hold or trim out.

## The world

An authored 160 km square — land, sea and mountains placed by hand rather than
generated randomly, so the map is the same every time.

| | Airport | Runways | Notes |
|---|---|---|---|
| **KMER** | Meridian International | 16L/34R, 16R/34L | Two parallels, CAT III, full approach lighting |
| **KHBR** | Harbor Bay Regional | 09/27 | Coastal, ILS |
| **KRDG** | Ridgeline Municipal | 12/30 | 4 200 ft elevation, short, no ILS, terrain on three sides |
| **KISV** | Isla Verde | 05/23 | Island, ocean off both ends |
| **KDFL** | Dustflat Field | 18/36, 13/31 grass | Small GA field |

Runway markings, touchdown zones, rubber deposits, PAPI and approach lighting
are laid out from real ICAO geometry, so they line up with where the aircraft
actually touches down. The PAPI changes from white to red as you go below the
beam, because it is computed from your position every frame.

## Systems

- **Autopilot** — HDG / NAV / LOC laterally, ALT / VS / FLC / GS vertically,
  plus autothrottle. Built as a proper cascade (vertical speed → flight-path
  angle → pitch attitude → elevator) with speed protection, so it gives up
  altitude before it gives up flying speed. It runs the trim to unload its own
  servo, refuses to engage outside a sane envelope, and disconnects on pilot
  override, stall, attitude limit or power loss. It will fly a full ILS from
  18 km out and hand over at minimums.
- **Failures** — engine failure and fire, fuel leak, hydraulic systems A and B,
  generator and total electrical, gear jam, flap asymmetry, pitot blockage,
  brake failure. Each has real consequences: an engine failure gives you
  asymmetric thrust to hold off, hydraulic loss slows the controls and locks the
  gear, electrical loss takes the autopilot and the glass instruments.
- **Bird strikes** — flocks below 10 000 ft, visible before they hit. What gets
  struck matters, and damage scales with closing speed.
- **Damage and crashes** — damage is tracked per component: each wing, the tail,
  the fuselage and each gear leg. It is not a switch between "fine" and
  "destroyed". Drag a wingtip on landing and that wing is wrecked, not the
  aeroplane: you lose most of the aileron on that side and gain a standing roll
  and yaw toward the damaged wing that you then have to hold off. Arrive hard
  enough and the legs fold and you slide; land gear-up and you scrape to a stop.
  Only a genuinely unsurvivable arrival — a steep, fast impact — writes the
  aircraft off outright, and the report then names what actually failed, with
  the numbers behind it. Land properly and you get scored instead.

## Controls

Mouse flies the yoke — click the canvas to capture the pointer. The axis is
shaped before it reaches the aircraft: a dead zone around neutral so it sits
still when you are not asking for anything, an expo curve so small corrections
are gentle while full deflection still reaches the stops, and a light filter to
keep mouse jitter out of the controls. All three are adjustable in Settings,
along with **Pitch direction** — *Yoke* (draw the mouse back to raise the nose,
like a control column) or *Camera* (the nose follows the cursor).

| | |
|---|---|
| Mouse | Pitch and roll |
| `W` / `S` | Throttle |
| `A` / `D` | Rudder |
| `Q` / `E` | Elevator trim |
| `G` · `F`/`V` · `B` | Gear · flaps · speedbrake |
| `Space` · `P` · `R` | Brakes · parking brake · reverse |
| `1`–`6`, `C` | Cockpit / chase / orbit / wing / tower / flyby |
| `Tab` · `K` · `O` | Autopilot · autothrottle · vertical mode |
| `[` `]` · `-` `=` | Heading select · altitude select |
| `N` · `I` | Cycle nav target · arm ILS approach |
| `Backspace` | Centre the yoke |
| `H` · `Esc` | Controls reference · pause |

Press `H` in flight for the full list.

## Performance

Four quality tiers; it probes the GPU on first run and picks one, and drops the
render scale if the frame rate stays low. The **volumetric clouds are by far the
most expensive thing here** — a full-screen ray-march — so they are the first
thing to turn off in Settings if you need frames. Shadow cascades and render
scale are next.

## Aircraft models

Everything is generated from parameters. Fuselages are lofted through stations
with independent half-width and half-height, so a body can be circular at the
cabin, flattened under the wing box and swept up at the tail. Lifting surfaces
come from a cambered NACA section with real sweep, taper, dihedral and washout,
and **every control surface is an actual slice cut from that same wing**, hinged
on its own swept hinge line rather than a slab laid on top — so the ailerons,
flaps, slats, spoilers, elevators and rudder all move as part of the structure.

The airliners carry a wing-body fairing, flap-track fairings, a dorsal fin
fillet, sharklets, an APU exhaust, and turbofans built from an inlet lip, fan
cowl, reverser cowl, core and exhaust plug with a spinning fan that blurs out as
N1 comes up. The Cessna gets a wrapped greenhouse, V lift struts with jury
struts, spring-steel main legs, wheel spats and a twisted tapering propeller.
An airliner is about 15 800 triangles, the Cessna about 7 000.

## A note on the textures

Every surface is painted onto a canvas at load time: asphalt with real aggregate
and hairline cracks, concrete slab joints, worn runway paint, rubber deposits,
panel lines and rivets, airline colour schemes. Nothing is downloaded. That keeps
the file self-contained and free of third-party image licensing, at the cost of a
second or two of start-up.

Airline schemes are simplified interpretations in each carrier's colours, not
reproductions of their trademarks.
