# SPITE-D on a TurtleBot

A trimmed build of the SPITE-D research package, cut down to what runs on
a robot: perception, prediction, certification, replanning and the three
ROS nodes. The roadmap generators, ablation tooling, simulation worlds
and test suite are not here, so there is no OMPL dependency and nothing
to generate before running.

Thanks for testing this. The short version of what you're looking at:

The robot plans using a **roadmap** — a precomputed graph of places it
could be and motions between them. That works fine until something
moves. This package watches the depth camera, tracks moving obstacles,
predicts where they're going, and marks roadmap edges as safe or blocked
*before* the obstacle gets there, so the plan can reroute early.

**The robot does not drive in this demo.** It sits still and watches.
Nothing here publishes to `cmd_vel`. Everything upstream of actually
turning the wheels is running; the execution loop isn't built yet, and
the roadmap is for a free-flying box rather than a differential drive, so
its paths aren't drivable as-is anyway.

What you should see: a person walking through the space appears as a
tracked box, roadmap edges near them turn red, and the planned path
jumps to a different route.

---

## 1. Build

```bash
mkdir -p ~/spited_ws/src && cd ~/spited_ws/src
git clone --recursive https://github.com/sai197161/turtlbot_spited_kernel.git
cd ~/spited_ws
source /opt/ros/jazzy/setup.bash
colcon build --packages-select spite_d
source install/setup.bash
```

If it complains that open-spite is missing, the `--recursive` didn't take:

```bash
cd ~/spited_ws/src/turtlbot_spited_kernel && git submodule update --init --recursive
```

Dependencies, if the build complains about missing headers:

```bash
sudo apt install libcgal-dev libeigen3-dev libfcl-dev \
                 nlohmann-json3-dev libopencv-dev
```

(OMPL is deliberately absent — roadmaps are not generated in this build.)

The build takes a couple of minutes the first time, since open-spite is
compiled from source along with it.

## 2. The roadmap

Nothing to do here — a roadmap is bundled with the package and the launch
file picks it up automatically. It covers a 4 m x 4 m area just above the
floor (1373 positions, ~13k motions between them), sized for a TurtleBot3
Waffle footprint.

If you need a different one — bigger area, fixed obstacles in the room,
denser graph — ask Sai; it is generated from the full research package,
which has the roadmap tools.

## 3. Find your depth topics

These vary by camera and driver, so check rather than assume:

```bash
ros2 topic list | grep -i depth
ros2 topic hz /camera/depth/image_raw     # or whatever yours is called
```

Note the image topic and its matching `camera_info` topic. You'll pass
both in below. Also check the encoding:

```bash
ros2 topic echo /camera/depth/image_raw --field encoding --once
```

`16UC1` means millimetres (use `depth_scale:=1000.0`, the default).
`32FC1` means metres and is converted automatically.

## 4. Place the camera

The roadmap is a 4 × 4 m square. The defaults assume the camera sits at
one edge of it, looking across:

```
        y
        ^      . . . . . . . .      roadmap slab, x,y in [-2, 2]
        |      . . . . . . . .
   cam  @----> . . . . . . . .
  (-2,0)   x   . . . . . . . .
```

So park the robot at one side of a roughly 4 m clear area, facing across
it. If you want to move it elsewhere, pass `camera_x`, `camera_y`,
`camera_z`, `camera_yaw` to match where it actually is — `camera_z` is
the camera's height off the floor, and `camera_yaw` is 0 when it faces
along +x.

## 5. Run it

```bash
ros2 launch spite_d turtlebot_demo.launch.py \
    depth_topic:=/camera/depth/image_raw \
    camera_info_topic:=/camera/depth/camera_info
```

RViz opens with the roadmap, tracked obstacles, and the planned path.
Then walk through the space in front of the robot.

---

## The numbers it prints

Both nodes print a summary line every 5 seconds. **These are the
preliminary results** — the main thing to copy out of the terminal and
send back.

```
[perception_node] perception | 14.8 Hz | detect 2.3 ms | tracks now 1 | distinct ids in 5s: 4
[validity_node]   validity | 1.2 Hz | certify 240.1 ms (obb 0.08 / tree 240.0) | edges red 412 gray 1877 of 13675 | replans 3
```

Reading them:

- **Hz** — how fast each stage is keeping up. Compare against the camera
  rate (`ros2 topic hz` on the depth topic). Perception should roughly
  match it; certification is the expensive stage and is expected to be
  slower. How much slower is exactly what we want to find out.
- **detect ms** — time to turn one depth image into tracked boxes.
- **distinct ids in 5s** — if one person is walking and this says 4, the
  tracker is losing and re-acquiring them. A known weakness; the number
  tells us how bad it is on real data.
- **certify ms (obb / tree)** — time to recertify the roadmap, split
  between building the obstacle's geometry and querying the roadmap. In
  simulation the tree query is essentially all of it; worth confirming.
- **edges red / gray** — how much of the roadmap the obstacle blocks.
  Red means provably blocked, gray means undecided. If nearly everything
  goes red with one person in the room, something is over-inflating.
- **replans** — how often the path changed.

Change the interval with `report_period_s:=2.0` if 5 s is too coarse.

---

## Checking it stage by stage

If something looks wrong, this narrows down *where*. Run each in a new
terminal with `source ~/spited_ws/install/setup.bash` first.

**Is depth arriving?**
```bash
ros2 topic hz /camera/depth/image_raw
```
Nothing → wrong topic name, or the camera driver isn't running.

**Is anything detected?**
```bash
ros2 topic echo /spite_d_perception/obstacles --once
```
Empty `obstacles: []` while someone is clearly standing there is the most
likely failure, and it's useful information — see tuning below.

**Are predictions being made?**
```bash
ros2 topic hz /spite_d_prediction/predictions
```

**Is the roadmap being certified?**
```bash
ros2 topic echo /spite_d_validity/path --once
```
The validity node also prints the roadmap size on startup.

**What's slow?** The terminal running `validity_node` is the one to
watch. Certification is the expensive part and we already know it's the
bottleneck — if the whole thing feels sluggish, that's expected and not
something you broke.

---

## Things that are likely to go wrong

**Nothing is detected.** The detector was written for simulated depth and
hasn't seen a real room before, so this is the most likely outcome and
the most useful thing to report. It histograms depth per image column and
looks for columns that agree, which finds upright objects against a
clear background. Real rooms have walls and furniture that also look like
upright objects. Worth trying:
- Stand 1.5–3 m away, clear of walls
- Check `min_dist`/`max_dist` cover your room's depth range
- `thresholdPoint` and `minLengthLine` control how strong a column has to
  be before it counts as an object — lower means more sensitive and more
  false positives

These aren't exposed as launch parameters yet. If detection is the
problem, say so and we'll add the knobs rather than you editing source.

**Obstacles appear in the wrong place.** Almost always the camera pose.
Check `camera_x/y/z/yaw` match where the robot actually is, and that
`camera_yaw` is 0 only if it genuinely faces +x.

**Track IDs change constantly.** Watch the number above each box in
RViz. If it jumps every second or two, the tracker is losing and
re-acquiring the same person. We know about this — it's on the list — but
how bad it is on real data is worth knowing.

**Everything is red.** Means the obstacle geometry is being inflated too
much or the roadmap sits in a place the person always occupies. Note
roughly what fraction of edges go red and we'll look at it.

**RViz is empty but topics are publishing.** Check the Fixed Frame is
`map` (top left of RViz).

---

## What's most useful to send back

Even a failure is a good result here — this is the first time any of it
has seen real hardware.

1. **The summary lines** from both terminals — a minute or two of them is
   enough. This is the actual result.
2. A rosbag, if you can — the next most valuable thing, because it lets
   everything be replayed offline without the robot:
   ```bash
   ros2 bag record /camera/depth/image_raw /camera/depth/camera_info \
                   /spite_d_perception/obstacles \
                   /spite_d_prediction/predictions \
                   /spite_d_validity/path
   ```
3. A screen recording of RViz, or just a photo of it
4. Roughly what you did — where you stood, how you walked, how far away,
   how many people were in view
