# Modular Navigation

Meta-package for Boeing's mobile robot navigation stack. Contains:
- `autonomy` - Main package for navigation
- `astar_planner` - Hybrid A* path planner
- `sim_band_planner` - Sim band (trajectory optimiser)
- `pure_pursuit_controller` - Pure Pursuit (controller)
- `gridmap` - Layered probability grid
- `hd_map` - interface package
- `map_manager` - map database interface

## Overview
Autonomy is a multi-layered system for 3D robot navigation (X,Y and rotation).
Conceptually, navigation can be broken into three layers:
1. Path planning - produce a rough path from A to B
2. Trajectory optimisation - Smooth out the path and swerve around immediate obstacles
3. Control - Follow the path and perform last-second collision checking

Each layer can in theory be swapped out with different algorithms without 
affecting the next. Therefore, the interface for each layer is separately 
defined in `navigation_interface`.

For now, there is only one of each layer:

## Path planning (Hybrid A*)
Hybrid A* divides the map into a grid and "explores" it by computing the 
cost required to get to each cell.

The cost is computed as the cost to get from Start to the cell, plus a
heuristic. The heuristic is an estimate of the cost to get from that cell
to the goal. Starting with the Start cell, the algorithm always explores 
the next cheapest cell. Because of the heuristic, the search may be "guided"
towards the goal without having to explore all the cells (as in Dijkstra),
significantly reducing the time required to find a valid path.

A well tuned heuristic is extremely important. If we underestimate the 
heuristic, it means each time we take a step towards the goal, the traversal
is more expensive than what the heuristic predicted, so we will stop exploring
down that path and end up doing a wide search (slow). However, an overestimating
heuristic will quickly guide you down a narrow path that could end up being
invalid (since it doesn't check collisions properly). Currently, the heuristic 
is calculated using a 2D Dijkstra (ignoring rotation) and estimating the
robot as a single circle. Interestingly, the Dijkstra is cached and done
in reverse (goal to robot) to be able to re-use the cached costs as the 
robot drives towards the goal.

The cost is also calculated differently for different types of motion.
For example, driving backwards is more expensive than forwards,
allowing us to tune the behaviour of the robot.

The robot's footprint is approximated using a few small circles. This makes
collision checking fast. For the heursitic, the robot uses a single conservative circle.

## Sim Band planner (trajectory optimisation)
http://www8.cs.umu.se/research/ifor/dl/Control/elastic%20bands.pdf

Trajectory optimisation does two things: smooth out the path generated by the planner and
alter the path slightly to swerve around obstacles. Sim band does this by treating the path 
like an elastic band with beads (the nodes). Obstacles exert forces on the nodes and 
we iteratively shift the nodes to achieve "equilibrium".

The footprint is also approximated using small circles like in the planner. This allows
us to calculate the force on each circle individually and from that, calculate the overall
torque on the robot, allowing us to optimise rotation as well as translation. Note that 
the footprint can be different from the planner's but in general, you should have 
a footprint that is equal to or smaller than the planner so that the planner will 
not generate a path that causes Sim Band to be in collision.

## Pure Pursuit Controller
The controller's job is to track the trajectory while performing last-second, thorough
collision checking. [Pure pursuit](https://www.ri.cmu.edu/pub_files/pub3/coulter_r_craig_1992_1/coulter_r_craig_1992_1.pdf) 
is one of the simplest control algorithms. The vehicle finds a point on the path that is
a certain distance ahead (the lookahead distance) and chases it. This method is popular
for Ackerman steered vehicles, but our robot can also pivot and strafe.

The main change to the traditional approach is to treat rotation as a third axis and
distance is measured as an L2 norm with an adjustable weight for the rotation.
Currently the weight is 1.0, meaning 1 rad is equivalent to 1m. We use this distance function
to find the point on the path that is closest to the robot as well as for calculating
the lookahead.

First, we find the closest node to the robot, then based on the robot's current speed,
we calculate the required lookahead distance by multiplying the speed with a lookahead time.
We then calculate the target by finding the first point on the path that is just ahead of
the required lookahead distance. The robot the chases this point with a PD controller.

When the robot approaches the final node, the integral term is enabled, allowing the 
robot to completely close the gap.

## How to build
`astar_planner` and `sim_band_planner` need to be built with compiler
optimisation to work in real time:
```bash
catkin build modular_navigation --cmake-args -DCMAKE_CXX_FLAGS="-O2 -Wall -Werror"
```

## Tests and debugging
### Testing the entire stack
Run the simulation and use `goal.py`. This script sends goal commands to
autonomy in a loop.

### astar_planner tests
Build astar_planner with the test arg:
```bash
catkin build astar_planner --make-args tests
```
The `unit_test` will be in `devel/.private/astar_planner/lib/astar_planner`

## Authors
The Boeing Company

Phillip Haeusler (Author)

William Ko (Maintainer - william.ko@boeing.com)

Leng Vongchanh

Alexandre Desbiez

Richard Bain

Jason Cochrane

Dominic Wierzbicki

Zhengzhi Zhang

## License
Copyright 2020 The Boeing Company

Licensed under the Apache License, Version 2.0 (the "License") with the following modification;
you may not use this file except in compliance with the Apache License and the following modification to it:

(Appended as Section 10)

By accepting this software, recipient agrees that the representations, warranties, obligations, and liabilities of The Boeing Company set forth in this software, if any, are exclusive and in substitution for all other all other representations, warranties, obligations, and liabilities of The Boeing Company.
Recipient hereby waives, releases, and renounces all other rights, remedies, and claims (including tortious claims for loss of or damage to property) of recipient against The Boeing Company with respect to this software.
The Boeing Company hereby disclaims all implied warranties, including but not limited to, all implied warranties of merchantability, fitness, course of dealing, and usage of trade.
The Boeing Company shall have no liability, whether arising in contract (including warranty), tort (whether or not arising from the negligence of The Boeing Company), or otherwise, for any loss of use, revenue, or profit, or for any other unspecified direct, indirect, incidental, or consequential damages for anything delivered or otherwise provided by The Boeing Company under this software.

You may obtain a copy of the original, unmodified License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

## Contributing
Any contribution that you make to this repository will
be under the Modified Apache 2 License, as dictated by that
[license](http://www.apache.org/licenses/LICENSE-2.0):

```
5. Submission of Contributions. Unless You explicitly state otherwise,
   any Contribution intentionally submitted for inclusion in the Work
   by You to the Licensor shall be under the terms and conditions of
   this License, without any additional terms or conditions.
   Notwithstanding the above, nothing herein shall supersede or modify
   the terms of any separate license agreement you may have executed
   with Licensor regarding such Contributions.
```

To contribute, issue a PR and @brta-mszarski (martin.a.szarski@boeing.com).