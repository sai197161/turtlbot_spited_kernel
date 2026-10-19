# SPITE-D on a TurtleBot

A pared-down build of the SPITE-D research package, cut to what runs on a
robot, for a first test on real hardware.

## What the full package does

A robot that plans with a **roadmap** — a precomputed graph of places it
could be and the motions between them — can answer planning queries very
quickly, because the expensive geometric work happened offline. That
holds until something moves. Once an obstacle walks through, some of
those precomputed motions are no longer safe, and finding out which ones
means re-checking robot geometry against obstacle geometry many times a
second.

SPITE-D watches a depth camera, tracks moving obstacles, predicts where
they are going, and marks roadmap edges as provably safe or provably
blocked *before* the obstacle arrives, so the plan can reroute early.
The research contribution is **spans**: freezing one prediction and
reusing its certification for as long as the obstacle keeps matching it,
instead of recomputing from scratch every frame.

## What this kernel tests

Whether the pipeline survives real-time data from real hardware, and
what it costs. The whole chain runs live:

```
depth image -> detect + track -> predict -> certify roadmap edges -> replan
```

The open questions, none of which have been answered outside simulation:

- Does the detector find people in a real room? It has only ever seen
  simulated depth, where there is no furniture and no clutter.
- Do track IDs stay stable on real sensor noise?
- Does certification produce sensible labels from real, noisy obstacle
  boxes?
- Can each stage keep up with the camera, and where does the time go?
- **Do spans actually hold up on real motion?** A span is only useful
  while the person keeps matching the prediction it froze. In
  simulation people walk in straight lines; real ones do not. How often
  a span survives on real tracking data is the number this is for.

That last one is the research contribution, and it is a switch:

```
use_spans:=true    spans: recertify only when the prediction breaks
use_spans:=false   baseline: recertify everything, every frame
```

Running the same walk both ways is the measurement — same room, same
camera, same roadmap, only the certification strategy differs. Please
do both if you have time; see "How to run" below.

**Not** in scope here, deliberately:

- **The robot does not drive.** Nothing publishes to `cmd_vel`. It sits
  still and watches; this is everything upstream of the wheels.
- **No roadmap generation, ablation tooling or test suite.** One
  roadmap ships prebuilt; the offline machinery stays in the research
  package.

## How to run

**1. Build.**

```bash
sudo apt install libcgal-dev libeigen3-dev libfcl-dev \
                 nlohmann-json3-dev libopencv-dev

mkdir -p ~/spited_ws/src && cd ~/spited_ws/src
git clone --recursive https://github.com/sai197161/turtlbot_spited_kernel.git
cd ~/spited_ws
source /opt/ros/jazzy/setup.bash
colcon build && source install/setup.bash
```

The first build takes a few minutes — a geometry library is compiled from
source alongside it. If it complains that open-spite is missing, the
`--recursive` did not take:
`cd src/turtlbot_spited_kernel && git submodule update --init --recursive`

A roadmap ships with the package, so there is nothing to generate.

**Heads up:** this package has not been compiled on a robot yet — it was
developed on a machine with no ROS, and the Linux box it normally builds
on is offline. A first build failing is a plausible outcome. If it does,
**send me the error rather than working around it** — it is far more
likely to be something I got wrong than something you need to fix, and
patching locally would put your copy out of sync with the package.

**2. Check it starts, before involving the robot.** This needs no
camera, no depth topic and no driving — it only proves the build is
sound and the roadmap loads:

```bash
ros2 run spite_d validity_node --ros-args \
  -p roadmap_graph:="$(ros2 pkg prefix spite_d)/share/spite_d/demo_roadmap/roadmap_graph.txt" \
  -p roadmap_geoms:="$(ros2 pkg prefix spite_d)/share/spite_d/demo_roadmap/roadmap_geoms.txt"
```

Expected: two log lines, then silence.

```
[INFO] [spite_d_validity]: certification mode: SPANS (amortised)
[INFO] [spite_d_validity]: roadmap loaded: 1373 vertices, 13675 edges; planning 0 -> 1
```

Then it sits there, idle, because nothing is publishing obstacles. Ctrl-C
out. If you see those two lines, the build, the install paths and the
parameters are all fine, and anything that goes wrong later is a
perception or hardware problem rather than a broken package — which is
the distinction worth having before you start chasing a detector that
will not fire.

Failures here, and what they mean:

- *`Package 'spite_d' not found`* — the workspace is not sourced in this
  terminal. `source ~/spited_ws/install/setup.bash`, in every terminal.
- *`error while loading shared libraries`* — the build installed the
  nodes but not their libraries. A package bug; send me the exact
  message.
- *`cannot read roadmap graph`* — the bundled roadmap did not get
  installed. Also a package bug.
- *Crash, or no output at all* — the most useful failure to report.
  Send the whole terminal.

**3. Find your depth topic.** These differ by camera and driver:

```bash
ros2 topic list | grep -i depth
ros2 topic hz /camera/depth/image_raw      # confirm it is publishing
```

**4. Park the robot.** The bundled roadmap covers a 4 m × 4 m area and
assumes the camera sits at one edge looking across it:

```
        y
        ^      . . . . . . . .     roadmap, x and y in [-2, 2]
        |      . . . . . . . .
   cam  @----> . . . . . . . .
  (-2,0)   x   . . . . . . . .
```

So put it at one side of roughly 4 m of clear floor, facing across. If
you put it somewhere else, pass `camera_x`, `camera_y`, `camera_z`
(height off the floor) and `camera_yaw` (0 when facing +x) to match.

**5. Launch the whole thing.**

```bash
ros2 launch spite_d turtlebot_demo.launch.py \
    depth_topic:=/camera/depth/image_raw \
    camera_info_topic:=/camera/depth/camera_info
```

RViz opens showing tracked obstacles, the roadmap, and the planned path.
Then walk through the space in front of the robot.

**6. Then run it again with spans off.** Same thing, plus
`use_spans:=false`:

```bash
ros2 launch spite_d turtlebot_demo.launch.py \
    depth_topic:=/camera/depth/image_raw \
    camera_info_topic:=/camera/depth/camera_info \
    use_spans:=false
```

Walk the same way for roughly the same length of time. Two runs of a
minute or two each is plenty. The pair of outputs is the result; either
run alone says much less.

## How to read the results

Both nodes print a summary every 5 seconds. **These lines are the
result** — a minute or two of them is what to send back.

```
perception | 14.8 Hz | detect 2.3 ms | tracks now 1 | distinct ids in 5s: 4
validity | 11.4 Hz | spans: 7 rebuilds / 57 obs (88% skipped), 12 expiries | certify 241.6 ms (obb 0.08 / tree 241.5) | edges red 436 gray 1902 of 13675 | replans 3
```

With `use_spans:=false` the middle field reads `baseline` instead, and
the Hz should drop a lot — that drop is the thing being measured.

**Hz** — whether each stage keeps up. Compare against the camera rate
from `ros2 topic hz`. Perception should roughly match it. Certification
is the expensive stage and is expected to lag; *how far* it lags is the
main thing being measured.

**detect ms** — time to turn one depth frame into tracked boxes.

**tracks now** — obstacles currently tracked. Should match the number of
people in view. Zero while someone is clearly standing there means the
detector is not firing, which is a plausible outcome and useful to know.

**distinct ids in 5s** — one person walking should produce one id. A
larger number means the tracker is losing and re-acquiring them. A known
weakness; this measures how bad it is on real data.

**spans: N rebuilds / M obs (P% skipped), E expiries** — the
contribution, in one field. Each frame every tracked obstacle is one
"obs". A rebuild is the expensive path: predict, build the inflated
geometry, re-query the roadmap trees. Everything else was absorbed by a
conformance check costing a handful of comparisons. So **P% is how much
certification work the spans avoided**.

A high P with the path still rerouting sensibly is the result we want. A
low P means real walking violates the frozen predictions too often for
spans to pay off, which is equally worth knowing and is the main risk
this run is testing. Expiries are slices of a span lapsing as time
passes — cheap healing, no geometry involved.

**certify ms (obb / tree)** — time to recertify the roadmap, split
between building the obstacle's geometry and querying the roadmap. In
simulation the tree query is essentially all of it, and confirming that
on hardware is one of the goals. Note this is time *per rebuild*, not
per frame, so it should be roughly the same in both runs — spans make it
happen less often, not faster.

**edges red / gray** — how much of the roadmap the obstacle blocks. Red
is provably blocked, gray is undecided. One person reddening most of the
roadmap would mean something is over-inflating.

**replans** — how often the path changed.

Use `report_period_s:=2.0` if 5 seconds is too coarse.

### What to send back

Even a failure is a useful result — this is the first time any of it has
touched real hardware.

1. **The summary lines** from both terminals, **for both runs** —
   spans on and spans off, labelled which is which.
2. **A rosbag**, if you can. This is what lets everything be replayed
   offline without the robot:
   ```bash
   ros2 bag record /camera/depth/image_raw /camera/depth/camera_info \
                   /spite_d_perception/obstacles \
                   /spite_d_prediction/predictions /spite_d_validity/path
   ```
3. A screen recording or photo of RViz.
4. Roughly what you did — where you stood, how you walked, how far from
   the camera, how many people were in view. If the two runs were
   noticeably different walks, say so; it affects how comparable the
   numbers are.

If something does not work, the node terminals plus which stage went
quiet is usually enough to pin it down. If step 2 above never printed its
two lines, stop there and send that — nothing downstream will mean
anything until the package builds and starts.
