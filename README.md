# SPITE-D on a TurtleBot

A stripped-down build of my SPITE-D research package for a first go on real hardware


## What this kernel tests


```
depth image -> detect + track -> predict -> certify roadmap edges -> replan
```


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

**Fair warning:** this has never actually been compiled on a robot. I
developed it on a machine with no ROS, and the Linux box I normally
build on is down right now. So the build failing is very possible.
If it does, please send me the error! although I'm super busy with school so not sure when I'll 
get around to everything...

**2. Check it even starts,** 

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

Ctrl-C out. If you got those two lines, the build and install paths and
parameters are all fine, so anything weird later is perception or
hardware issue.

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

So please stick it at one side of roughly 4 m of clear floor, facing across. If
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
result**, and what I would like back.

```
perception | 14.8 Hz | detect 2.3 ms | tracks now 1 | distinct ids in 5s: 4
validity | 11.4 Hz | spans: 7 rebuilds / 57 obs (88% skipped), 12 expiries | certify 241.6 ms (obb 0.08 / tree 241.5) | edges red 436 gray 1902 of 13675 | replans 3
```

With `use_spans:=false` that middle field says `baseline` instead, and
the Hz should (ideally) drop.

**Hz** — Compare to the camera rate from
`ros2 topic hz`. 

**detect ms** — how long one depth frame takes to turn into boxes.

**tracks now** — obstacles currently tracked. Should match how many
people are in view. 

**distinct ids in 5s** — one person walking should give one id. More
than that means the tracker keeps losing and re-finding them.

**spans: N rebuilds / M obs (P% skipped), E expiries** — Every tracked obstacle in every frame counts
as one "obs". A rebuild is expensive. predict, build the
inflated geometry, re-query the roadmap trees. else conformance check that's a handful of comparisons. So
P% is how much certification work the spans saved.

High P with the path still rerouting sensibly is what I'm hoping for.

**certify ms (obb / tree)** — time to recertify the roadmap, split
between building the obstacle geometry and querying the roadmap. In
simulation the tree query is basically all of it.

**edges red / gray** — how much of the roadmap the obstacle blocks. Red
is provably blocked, gray is undecided.

**replans** — how often the path changed.

Pass `report_period_s:=2.0` if 5 seconds feels too coarse.

### What to send back

Even a total failure is fine.

1. **The summary lines** from both terminals, **for both runs**, and
   which is which.
2. **A rosbag** lets me
   replay everything offline without the robot:
   ```bash
   ros2 bag record /camera/depth/image_raw /camera/depth/camera_info \
                   /spite_d_perception/obstacles \
                   /spite_d_prediction/predictions /spite_d_validity/path
   ```
3. A screen recording or even just a photo of RViz.
4. Rough obstacle behavior.

Thanks again!
