# SPITE-D on a TurtleBot

A stripped-down build of my SPITE-D research package — just the parts
that run on a robot — for a first go on real hardware. Thanks for
testing it!

## What the full package does

If a robot plans using a **roadmap** (a precomputed graph of places it
can be and the motions between them), planning queries are really fast,
because all the expensive geometry happened offline. That's great right
up until something moves. As soon as someone walks through, some of
those precomputed motions aren't safe anymore, and figuring out which
ones means re-checking the robot's geometry against the obstacle's many
times a second.

SPITE-D watches a depth camera, tracks the moving obstacles, predicts
where they're headed, and marks roadmap edges as provably safe or
provably blocked *before* the obstacle gets there, so the plan can
reroute early. The research bit is **spans**: freeze one prediction and
keep reusing its certification for as long as the obstacle keeps
matching it, instead of redoing everything from scratch every frame.

## What this kernel tests

Basically: does any of this survive real data from real hardware, and
what does it cost? The whole chain runs live:

```
depth image -> detect + track -> predict -> certify roadmap edges -> replan
```

Things I genuinely don't know the answer to, because it's only ever run
in simulation:

- Will the detector find people in an actual room? It's only ever seen
  simulated depth, where there's no furniture and no clutter.
- Do track IDs stay stable once there's real sensor noise?
- Does certification give sensible labels from real, noisy boxes?
- Can each stage keep up with the camera, and where's the time going?
- **Do spans hold up on real motion?** A span is only useful while the
  person keeps matching the prediction it froze. Simulated people walk
  in straight lines. Real ones don't. How often a span survives on real
  tracking data is the number I'm really after.

That last one's the contribution, and it's just a flag:

```
use_spans:=true    spans: only recertify when the prediction breaks
use_spans:=false   baseline: recertify everything, every frame
```

So if you run the same walk both ways, that's the measurement — same
room, same camera, same roadmap, only the strategy changes. Would be
great if you can do both; steps 5 and 6 below.

A couple of things it deliberately doesn't do:

- **The robot doesn't drive.** Nothing publishes to `cmd_vel`. It just
  sits there and watches. This is everything upstream of the wheels.
- **No roadmap generation, ablation tooling or tests.** A roadmap ships
  prebuilt; all the offline machinery stayed in the research package.

## How to run

**1. Build it.**

```bash
sudo apt install libcgal-dev libeigen3-dev libfcl-dev \
                 nlohmann-json3-dev libopencv-dev

mkdir -p ~/spited_ws/src && cd ~/spited_ws/src
git clone --recursive https://github.com/sai197161/turtlbot_spited_kernel.git
cd ~/spited_ws
source /opt/ros/jazzy/setup.bash
colcon build && source install/setup.bash
```

First build takes a few minutes — there's a geometry library compiled
from source alongside it. If it says open-spite is missing, the
`--recursive` didn't take:
`cd src/turtlbot_spited_kernel && git submodule update --init --recursive`

A roadmap ships with the package, so there's nothing to generate.

**Fair warning:** this has never actually been compiled on a robot. I
developed it on a machine with no ROS, and the Linux box I normally
build on is down right now. So the build failing is a real possibility.
If it does, **just send me the error instead of working around it** —
it's way more likely to be something I got wrong than something you
need to fix, and patching it locally would put your copy out of sync.

**2. Check it even starts, before you bother with the robot.** No
camera, no depth topic, no driving — this just proves the build works
and the roadmap loads:

```bash
ros2 run spite_d validity_node --ros-args \
  -p roadmap_graph:="$(ros2 pkg prefix spite_d)/share/spite_d/demo_roadmap/roadmap_graph.txt" \
  -p roadmap_geoms:="$(ros2 pkg prefix spite_d)/share/spite_d/demo_roadmap/roadmap_geoms.txt"
```

You should get two lines and then nothing:

```
[INFO] [spite_d_validity]: certification mode: SPANS (amortised)
[INFO] [spite_d_validity]: roadmap loaded: 1373 vertices, 13675 edges; planning 0 -> 1
```

Then it just sits there, because nothing's publishing obstacles yet.
Ctrl-C out. If you got those two lines, the build and install paths and
parameters are all fine — so anything weird later is perception or
hardware, not a broken package. Worth knowing *before* you start
chasing a detector that won't fire.

If it goes wrong here:

- *`Package 'spite_d' not found`* — workspace isn't sourced in that
  terminal. `source ~/spited_ws/install/setup.bash`. Every terminal,
  annoyingly.
- *`error while loading shared libraries`* — build installed the nodes
  but not their libraries. That's my bug, send me the message.
- *`cannot read roadmap graph`* — the bundled roadmap didn't install.
  Also my bug.
- *Crashes, or prints nothing at all* — honestly the most useful thing
  you could send me. Paste the whole terminal.

**3. Find your depth topic.** Varies by camera and driver:

```bash
ros2 topic list | grep -i depth
ros2 topic hz /camera/depth/image_raw      # check it's actually publishing
```

**4. Park the robot.** The bundled roadmap covers a 4 m × 4 m area and
assumes the camera's at one edge looking across it:

```
        y
        ^      . . . . . . . .     roadmap, x and y in [-2, 2]
        |      . . . . . . . .
   cam  @----> . . . . . . . .
  (-2,0)   x   . . . . . . . .
```

So stick it at one side of roughly 4 m of clear floor, facing across. If
you put it somewhere else, pass `camera_x`, `camera_y`, `camera_z`
(height off the floor) and `camera_yaw` (0 = facing +x) to match.

**5. Run the whole thing.**

```bash
ros2 launch spite_d turtlebot_demo.launch.py \
    depth_topic:=/camera/depth/image_raw \
    camera_info_topic:=/camera/depth/camera_info
```

RViz opens with the tracked obstacles, the roadmap and the planned path.
Then go walk around in front of the robot.

**6. Now do it again with spans off.** Same command plus
`use_spans:=false`:

```bash
ros2 launch spite_d turtlebot_demo.launch.py \
    depth_topic:=/camera/depth/image_raw \
    camera_info_topic:=/camera/depth/camera_info \
    use_spans:=false
```

Try to walk roughly the same way for roughly the same amount of time. A
minute or two each is plenty. It's the *pair* of runs that's the result
— either one on its own doesn't say much.

## How to read the results

Both nodes print a summary line every 5 seconds. **Those lines are the
result** — a minute or two of them is what I need back.

```
perception | 14.8 Hz | detect 2.3 ms | tracks now 1 | distinct ids in 5s: 4
validity | 11.4 Hz | spans: 7 rebuilds / 57 obs (88% skipped), 12 expiries | certify 241.6 ms (obb 0.08 / tree 241.5) | edges red 436 gray 1902 of 13675 | replans 3
```

With `use_spans:=false` that middle field says `baseline` instead, and
the Hz should drop a lot. That drop is the whole point.

**Hz** — whether each stage keeps up. Compare to the camera rate from
`ros2 topic hz`. Perception should roughly match it. Certification is
the expensive stage and will lag; *how much* it lags is the main thing
I'm measuring.

**detect ms** — how long one depth frame takes to turn into boxes.

**tracks now** — obstacles currently tracked. Should match how many
people are in view. Zero while someone's obviously standing right there
means the detector isn't firing, which is a totally plausible outcome
and useful to know.

**distinct ids in 5s** — one person walking should give one id. More
than that means the tracker keeps losing and re-finding them. Known
weakness; this tells me how bad it is on real data.

**spans: N rebuilds / M obs (P% skipped), E expiries** — the
contribution in one field. Every tracked obstacle in every frame counts
as one "obs". A rebuild is the expensive path: predict, build the
inflated geometry, re-query the roadmap trees. Everything else got
absorbed by a conformance check that's a handful of comparisons. So
**P% is how much certification work the spans saved**.

High P with the path still rerouting sensibly is what I'm hoping for.
Low P means real walking breaks the frozen predictions too often for
spans to be worth it — which is honestly just as useful to find out,
and it's the main risk here. Expiries are span slices lapsing as time
passes; cheap, no geometry involved.

**certify ms (obb / tree)** — time to recertify the roadmap, split
between building the obstacle geometry and querying the roadmap. In
simulation the tree query is basically all of it, and confirming that on
real hardware is one of the goals. This is per *rebuild*, not per frame,
so it should be about the same in both runs — spans make it happen less
often, not faster.

**edges red / gray** — how much of the roadmap the obstacle blocks. Red
is provably blocked, gray is undecided. One person turning most of the
roadmap red would mean something's over-inflating.

**replans** — how often the path changed.

Pass `report_period_s:=2.0` if 5 seconds feels too coarse.

### What to send back

Even a total failure is a useful result here — this is the first time
any of it has touched real hardware.

1. **The summary lines** from both terminals, **for both runs**, and
   which is which.
2. **A rosbag** if you can manage it. This is the big one — it lets me
   replay everything offline without the robot:
   ```bash
   ros2 bag record /camera/depth/image_raw /camera/depth/camera_info \
                   /spite_d_perception/obstacles \
                   /spite_d_prediction/predictions /spite_d_validity/path
   ```
3. A screen recording or even just a photo of RViz.
4. Roughly what you did — where you stood, how you walked, how far from
   the camera, how many people were in view. And if the two runs ended
   up being pretty different walks, mention it, since that affects how
   comparable the numbers are.

If something's broken, the node terminals plus which stage went quiet is
usually enough for me to track it down. And if step 2 never printed its
two lines, just stop there and send me that — nothing after it will mean
anything until the package actually builds.
