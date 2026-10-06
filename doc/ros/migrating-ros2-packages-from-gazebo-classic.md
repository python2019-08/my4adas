# Migrating ROS 2 packages that use Gazebo Classic

[\#](https://gazebosim.org/docs/harmonic/migrating_gazebo_classic_ros2_packages/)

The Gazebo simulator has its roots in the Gazebo Classic project, but it
has a few significant differences that affect how a ROS 2 project uses
the simulator. One difference is that ROS 2 projects now use the
[ros_gz](https://github.com/gazebosim/ros_gz){.reference .external}
package instead of
[gazebo_ros_pkgs](https://github.com/ros-simulation/gazebo_ros_pkgs){.reference
.external} as the source of launch files and other useful utilities.
Another major difference is that while
[gazebo_ros_pkgs](https://github.com/ros-simulation/gazebo_ros_pkgs){.reference
.external} provided a set of plugins that directly get loaded by Gazebo
Classic and run as part of the simulation to provide an interface
between ROS and Gazebo Classic,
[ros_gz](https://github.com/gazebosim/ros_gz){.reference .external} is
primarily used as a bridge between ROS and gz-transport topics. Knowing
these conceptual differences is important in making the transition.

**Note:** Since the name of the project has gone through two major
changes, we highly recommend you read the
[history](https://gazebosim.org/about){.reference .external} of the
project to have a better understanding of the terminology used in this
tutorial and elsewhere. As a convention we refer to older versions of
Gazebo, those with release numbers like Gazebo 9 and Gazebo 11 as
"Gazebo Classic." Newer versions of Gazebo, formerly called "Ignition",
with lettered releases names like Harmonic, are referred to as just
"Gazebo" in this document. This tutorial will show how to migrate an
existing ROS 2 package that uses the [`gazebo_ros_pkgs`{.docutils
.literal .notranslate}]{.pre} package to the new [`ros_gz`{.docutils
.literal .notranslate}]{.pre}. We will use the
[turtlebot3_simulations](https://github.com/ROBOTIS-GIT/turtlebot3_simulations/){.reference
.external} package as an example. The complete, migrated version of
[`turtlebot3_simulations`{.docutils .literal .notranslate}]{.pre}
covered in this tutorial, can be found in [this
fork](https://github.com/azeey/turtlebot3_simulations/tree/new_gazebo){.reference
.external}.

We'll start by following the [PC
Setup](https://emanual.robotis.com/docs/en/platform/turtlebot3/quick-start/){.reference
.external} guide to install the necessary prerequisites for simulating
Turtlebot3. This will install additional packages, such as
[Nav2](https://github.com/ros-planning/navigation2){.reference
.external} and
[Cartographer](https://github.com/cartographer-project/cartographer){.reference
.external}, which we will be using later on in this tutorial, so make
sure to not skip this step.

The next step is to clone the [`turtlebot3_simulation`{.docutils
.literal .notranslate}]{.pre} package. We'll use the
[`humble-devel`{.docutils .literal .notranslate}]{.pre} branch, which at
the time of writing had a SHA of [`d16cdbe`{.docutils .literal
.notranslate}]{.pre}
 
```sh
source /opt/ros/humble/setup.bash
mkdir -p ~/turtlebot3_ws/src
cd ~/turtlebot3_ws/src
git clone -b humble-devel https://github.com/ROBOTIS-GIT/turtlebot3_simulations/
```
 

Install dependencies using [`rosdep`{.docutils .literal
.notranslate}]{.pre}
 
``` sh
sudo rosdep init # only needed if  using rosdep
rosdep install --from-paths . -i -y
```

  
Finally, build the project and check that the Gazebo classic simulation
works. (See the [Gazebo Simulation
guide](https://emanual.robotis.com/docs/en/platform/turtlebot3/simulation/#gazebo-simulation){.reference
.external})
 
```sh
cd ~/turtlebot3_ws
colcon build --symlink-install
source ~/turtlebot3_ws/install/setup.bash

export TURTLEBOT3_MODEL=waffle
ros2 launch turtlebot3_gazebo empty_world.launch.py
```
 

Here's a screenshot of Turtlebot3 running in Gazebo Classic obtained by
launching [`empty_world.launch.py`{.docutils .literal
.notranslate}]{.pre}.

![Screenshot of turtlebot 3 running in Gazebo
Classic](./migrating-ros2-packages-from-gazebo-classic_files/gazebo_classic_turtlebot3.png)

Once, we're sure that the Gazebo classic simulation is running properly,
we create a new branch in which we'll make the changes to migrate to the
new Gazebo. 
```sh
git checkout -b new_gazebo
```

 

The changes we need to make are:

1.  Modify [`package.xml`{.docutils .literal .notranslate}]{.pre} and
    [`CMakeLists.txt`{.docutils .literal .notranslate}]{.pre} files
    replacing [`gazebo`{.docutils .literal .notranslate}]{.pre},
    [`gazebo_ros_pkgs`{.docutils .literal .notranslate}]{.pre}, etc with
    packages from [`ros_gz`{.docutils .literal .notranslate}]{.pre}.

2.  Edit launch files that start Gazebo (e.g.
    [`empty_world.launch.py`{.docutils .literal .notranslate}]{.pre})

3.  Update the world SDFormat file.

4.  Edit launch files that spawn models.

5.  Edit model SDFormat files.

6.  Bridge ROS topics.
 
## Update package dependencies 

The turtlebot 3 package depends on [`gazebo_ros_pkgs`{.docutils .literal
.notranslate}]{.pre}, which is the package that provides launch files,
plugins, and other utilities for using Gazebo classic with ROS 2. The
equivalent for the new Gazebo is [`ros_gz`{.docutils .literal
.notranslate}]{.pre}, but [`ros_gz`{.docutils .literal
.notranslate}]{.pre} is actually a meta-package that contains a few
packages. It's okay to replace [`gazebo_ros_pkgs`{.docutils .literal
.notranslate}]{.pre} with [`ros_gz`{.docutils .literal
.notranslate}]{.pre} here, but using just the subset of packages needed
for your project will reduce the number of dependencies. For the
[`turtlebot3_simulation`{.docutils .literal .notranslate}]{.pre}
package, we will only need [`ros_gz_bridge`{.docutils .literal
.notranslate}]{.pre}, [`ros_gz_image`{.docutils .literal
.notranslate}]{.pre}, and [`ros_gz_sim`{.docutils .literal
.notranslate}]{.pre} for now. [`ros_gz_bridge`{.docutils .literal
.notranslate}]{.pre} and [`ros_gz_image`{.docutils .literal
.notranslate}]{.pre} provide topic bridges between Gazebo and ROS while
[`ros_gz_sim`{.docutils .literal .notranslate}]{.pre} provides launch
files and other utilities that help with starting Gazebo and spawning
models.

After making the change, lines 17-21 of [`package.xml`{.docutils
.literal .notranslate}]{.pre} will look like this:
 
```xml
...
  <depend>geometry_msgs</depend>
  <depend>nav_msgs</depend>
  <depend>rclcpp</depend>
  <depend>ros_gz_bridge</depend>
  <depend>ros_gz_image</depend>
  <depend>ros_gz_sim</depend>
  <depend>sensor_msgs</depend>
  <depend>tf2</depend>
...
```
 

You can find the [Gazebo Classic package XML file
here](https://github.com/ROBOTIS-GIT/turtlebot3_simulations/blob/d16cdbe7ecd601ccad48f87f77b6d89079ec5ac1/turtlebot3_gazebo/package.xml){.reference
.external}, and the [updated Gazebo package XML file
here](https://github.com/azeey/turtlebot3_simulations/blob/new_gazebo/turtlebot3_gazebo/package.xml){.reference
.external}.

After making the change, we'll need to install the new dependencies. The
following command will automatically install the necessary Gazebo
version.
 
```sh
rosdep install --from-paths . -i -y
```
 

This tutorial assumes you are using Gazebo Fortress as it is the version
of Gazebo officially paired with ROS 2 Humble. While it is possible to
use newer versions of Gazebo with ROS 2 Humble, it requires extra work
and is not recommend for most users. See [[Installing Gazebo with
ROS]{.doc .std
.std-doc}](https://gazebosim.org/docs/harmonic/ros_installation/){.reference
.internal} to learn more. If you intend to switch back and forth between
the new Gazebo and Gazebo Classic, it's best to use Gazebo Fortress
since the newer versions will automatically uninstall Gazebo Classic.
With that being said, the concepts covered by the tutorial should work
with newer versions of ROS 2 and Gazebo.
 
## Launch the world 

We now need to edit
[`turtlebot3_gazebo/launch/empty_world.launch.py`{.docutils .literal
.notranslate}]{.pre} and replace any use of [`gazebo_ros_pkgs`{.docutils
.literal .notranslate}]{.pre}. You can find the [Gazebo Classic empty
world launch file before editing
here](https://github.com/ROBOTIS-GIT/turtlebot3_simulations/blob/d16cdbe7ecd601ccad48f87f77b6d89079ec5ac1/turtlebot3_gazebo/launch/empty_world.launch.py){.reference
.external}, and the [updated Gazebo empty world launch file
here](https://github.com/azeey/turtlebot3_simulations/blob/new_gazebo/turtlebot3_gazebo/launch/empty_world.launch.py){.reference
.external}. First replace the call to
[`get_package_share_directory`{.docutils .literal .notranslate}]{.pre}
to find [`ros_gz_sim`{.docutils .literal .notranslate}]{.pre}. The code
will change from:

::: {.highlight-python .notranslate}
::: highlight
``` {#codecell6 tabindex="-1"}
pkg_gazebo_ros = get_package_share_directory('gazebo_ros')
```
 

to: 
```py
ros_gz_sim = get_package_share_directory('ros_gz_sim')
```
  
Next, change [`gzserver_cmd`{.docutils .literal .notranslate}]{.pre} to
use [`ros_gz_sim`{.docutils .literal .notranslate}]{.pre}

::: {.highlight-python .notranslate}
::: highlight
``` {#codecell8 tabindex="-1"}
gzserver_cmd = IncludeLaunchDescription(
    PythonLaunchDescriptionSource(
        os.path.join(ros_gz_sim, 'launch', 'gz_sim.launch.py')
    ),
    launch_arguments={'gz_args': ['-r -s -v4 ', world], 'on_exit_shutdown': 'true'}.items()
)
```
 
This uses the [`gz_sim.launch.py`{.docutils .literal
.notranslate}]{.pre} launch file from the [`ros_gz_sim`{.docutils
.literal .notranslate}]{.pre} package. The launch file takes the
[`gz_args`{.docutils .literal .notranslate}]{.pre} argument which is a
list of command line flags that will be passed to [`ign`{.docutils
.literal .notranslate}]{.pre}` `{.docutils .literal
.notranslate}[`gazebo`{.docutils .literal .notranslate}]{.pre}
([`gz`{.docutils .literal .notranslate}]{.pre}` `{.docutils .literal
.notranslate}[`sim`{.docutils .literal .notranslate}]{.pre} in Garden
and later). [`-s`{.docutils .literal .notranslate}]{.pre} causes only
the Gazebo server to run without the GUI client and [`-r`{.docutils
.literal .notranslate}]{.pre} tells Gazebo to start running simulation
immediately. Lastly, we are using the [`-v4`{.docutils .literal
.notranslate}]{.pre} flag which sets the verbosity level of Gazebo's
console output.

**Note:** The list assigned to [`gz_args`{.docutils .literal
.notranslate}]{.pre} is concatenated into a string with code equivalent
to [`''.join(gz_args)`{.docutils .literal .notranslate}]{.pre}, so it's
important to keep whitespace where necessary. Note the space after 4 in
[`'-v4`{.docutils .literal .notranslate}]{.pre}` `{.docutils .literal
.notranslate}[`'`{.docutils .literal .notranslate}]{.pre}.

The [`world`{.docutils .literal .notranslate}]{.pre} argument will be
substituted by [`launch`{.docutils .literal .notranslate}]{.pre} before
running Gazebo. In this launch file, [`world`{.docutils .literal
.notranslate}]{.pre} is a python variable, so it is possible to use
python string formatting: [`gz_args:`{.docutils .literal
.notranslate}]{.pre}` `{.docutils .literal
.notranslate}[`f'-s`{.docutils .literal
.notranslate}]{.pre}` `{.docutils .literal .notranslate}[`-r`{.docutils
.literal .notranslate}]{.pre}` `{.docutils .literal
.notranslate}[`-v4`{.docutils .literal .notranslate}]{.pre}` `{.docutils
.literal .notranslate}[`{world}'`{.docutils .literal
.notranslate}]{.pre}. But if we wanted to use a
[`LaunchConfiguration`{.docutils .literal .notranslate}]{.pre} variable
for [`world`{.docutils .literal .notranslate}]{.pre}, we will need to
use a list so that [`launch`{.docutils .literal .notranslate}]{.pre}
will make the substitution for us.

The [`on_exit_shutdown`{.docutils .literal .notranslate}]{.pre} argument
ensures that if the Gazebo server exits for any reason, the rest of the
nodes in the launch file are shutdown

The GUI client is launched in a similar way, but we change
[`gz_args`{.docutils .literal .notranslate}]{.pre} to [`-g`{.docutils
.literal .notranslate}]{.pre} to run just the GUI client.

 
```py
gzclient_cmd = IncludeLaunchDescription(
    PythonLaunchDescriptionSource(
        os.path.join(ros_gz_sim, 'launch', 'gz_sim.launch.py')
    ),
    launch_arguments={'gz_args': '-g -v4 '}.items()
)
```
 
Finally, we need to set the environment variable
[`GZ_SIM_RESOURCE_PATH`{.docutils .literal .notranslate}]{.pre} so
Gazebo can know where to find models. See the [Finding
resource](https://gazebosim.org/api/sim/8/resources.html){.reference
.external} document to learn more about this environment variable. This
was not needed for [`gazebo_ros_pkgs`{.docutils .literal
.notranslate}]{.pre} because it used the [`<export>`{.docutils .literal
.notranslate}]{.pre} tag in [`package.xml`{.docutils .literal
.notranslate}]{.pre} to populate a similar environment variable for
Gazebo ([`GAZEBO_MODEL_PATH`{.docutils .literal .notranslate}]{.pre}).

First, import [`AppendEnvironmentVariable`{.docutils .literal
.notranslate}]{.pre}
 
```py
from launch.actions import AppendEnvironmentVariable
```
 

and create a [`launch`{.docutils .literal .notranslate}]{.pre} action
that appends the environment variable with the location of the
[`models`{.docutils .literal .notranslate}]{.pre} directory in
[`turtlebot3_gazebo`{.docutils .literal .notranslate}]{.pre}.
 
```py
set_env_vars_resources = AppendEnvironmentVariable(
        'GZ_SIM_RESOURCE_PATH',
        os.path.join(get_package_share_directory('turtlebot3_gazebo'),
                     'models'))
```
 

We'll then need to add the action to [`ld`{.docutils .literal
.notranslate}]{.pre}, the [`LaunchDescription`{.docutils .literal
.notranslate}]{.pre} variable returned by
[`generate_launch_description`{.docutils .literal .notranslate}]{.pre}.
 
```py
ld.add_action(set_env_vars_resources)
```
 
We are now ready to test the launch file. Comment out
[`ld.add_action(spawn_turtlebot_cmd)`{.docutils .literal
.notranslate}]{.pre} and run:
 
```sh
ros2 launch turtlebot3_gazebo empty_world.launch.py
```
 

**More than likely, this will fail** because Gazebo could not find
models referenced in the world SDFormat file. The next step is to fix
that.

**Note:** Due to a bug in the GUI client, there might be a lingering
[`ign`{.docutils .literal .notranslate}]{.pre}` `{.docutils .literal
.notranslate}[`gazebo`{.docutils .literal
.notranslate}]{.pre}` `{.docutils .literal .notranslate}[`-g`{.docutils
.literal .notranslate}]{.pre} or [`gz`{.docutils .literal
.notranslate}]{.pre}` `{.docutils .literal .notranslate}[`sim`{.docutils
.literal .notranslate}]{.pre}` `{.docutils .literal
.notranslate}[`-g`{.docutils .literal .notranslate}]{.pre} process after
terminating the launch. You can kill it using [`pkill`{.docutils
.literal .notranslate}]{.pre}` `{.docutils .literal
.notranslate}[`-f`{.docutils .literal .notranslate}]{.pre}` `{.docutils
.literal .notranslate}[`-9`{.docutils .literal
.notranslate}]{.pre}` `{.docutils .literal
.notranslate}[`'ign`{.docutils .literal
.notranslate}]{.pre}` `{.docutils .literal
.notranslate}[`gazebo'`{.docutils .literal .notranslate}]{.pre}
 
## Edit world SDFormat file 

The file we will be editing is
[`turtlebot3_gazebo/worlds/empty_world.world`{.docutils .literal
.notranslate}]{.pre}. This file references the models [`sun`{.docutils
.literal .notranslate}]{.pre} and [`ground_plane`{.docutils .literal
.notranslate}]{.pre}. In Gazebo Classic, these models were either
shipped with the simulator or downloaded from the gazebo [model
repository](https://github.com/osrf/gazebo_models){.reference
.external}. The new Gazebo does not ship these models, instead we can
use models from Fuel or add the models directly to the world file.

To use fuel models, replace the [`include`{.docutils .literal
.notranslate}]{.pre} tags for [`sun`{.docutils .literal
.notranslate}]{.pre} and [`ground_place`{.docutils .literal
.notranslate}]{.pre} with

 
```xml
<include>
  <uri>
    https://fuel.gazebosim.org/1.0/OpenRobotics/models/Ground Plane
  </uri>
</include>

<include>
  <uri>
    https://fuel.gazebosim.org/1.0/OpenRobotics/models/Sun
  </uri>
</include>
``` 

For reference, you can find the [Gazebo Classic empty world
here](https://github.com/ROBOTIS-GIT/turtlebot3_simulations/blob/d16cdbe7ecd601ccad48f87f77b6d89079ec5ac1/turtlebot3_gazebo/worlds/empty_world.world){.reference
.external}, and the [updated new Gazebo empty world
here](https://github.com/azeey/turtlebot3_simulations/blob/new_gazebo/turtlebot3_gazebo/worlds/empty_world.world){.reference
.external}. Relaunching [`empty_world.launch.py`{.docutils .literal
.notranslate}]{.pre} should now start the simulator successfully.
 
## Spawn model 

In this step, we will modify
[`turtlebot3_gazebo/launch/spawn_turtlebot3.launch.py`{.docutils
.literal .notranslate}]{.pre}. Again, we need to change
[`gazebo_ros`{.docutils .literal .notranslate}]{.pre} to
[`ros_gz_sim`{.docutils .literal .notranslate}]{.pre}. We'll also need
to change [`spawn_entity.py`{.docutils .literal .notranslate}]{.pre} to
[`create`{.docutils .literal .notranslate}]{.pre}, which is the node in
[`ros_gz_sim`{.docutils .literal .notranslate}]{.pre} that provides
model spawning functionality. From the argument list,
[`-entity`{.docutils .literal .notranslate}]{.pre} needs to be replaced
with [`-name`{.docutils .literal .notranslate}]{.pre}. You can run
[`ros2`{.docutils .literal .notranslate}]{.pre}` `{.docutils .literal
.notranslate}[`run`{.docutils .literal .notranslate}]{.pre}` `{.docutils
.literal .notranslate}[`ros_gz_sim`{.docutils .literal
.notranslate}]{.pre}` `{.docutils .literal
.notranslate}[`create`{.docutils .literal
.notranslate}]{.pre}` `{.docutils .literal
.notranslate}[`--helpshort`{.docutils .literal .notranslate}]{.pre} to
see more options.

The resulting [`Node`{.docutils .literal .notranslate}]{.pre} should
look like:
 
```py
start_gazebo_ros_spawner_cmd = Node(
    package='ros_gz_sim',
    executable='create',
    arguments=[
        '-name', TURTLEBOT3_MODEL,
        '-file', urdf_path,
        '-x', x_pose,
        '-y', y_pose,
        '-z', '0.01'
    ],
    output='screen',
)
```

![](data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIGNsYXNzPSJpY29uIGljb24tdGFibGVyIGljb24tdGFibGVyLWNvcHkiIHdpZHRoPSI0NCIgaGVpZ2h0PSI0NCIgdmlld2JveD0iMCAwIDI0IDI0IiBzdHJva2Utd2lkdGg9IjEuNSIgc3Ryb2tlPSIjMDAwMDAwIiBmaWxsPSJub25lIiBzdHJva2UtbGluZWNhcD0icm91bmQiIHN0cm9rZS1saW5lam9pbj0icm91bmQiPgogIDx0aXRsZT5Db3B5IHRvIGNsaXBib2FyZDwvdGl0bGU+CiAgPHBhdGggc3Ryb2tlPSJub25lIiBkPSJNMCAwaDI0djI0SDB6IiBmaWxsPSJub25lIj48L3BhdGg+CiAgPHJlY3QgeD0iOCIgeT0iOCIgd2lkdGg9IjEyIiBoZWlnaHQ9IjEyIiByeD0iMiI+PC9yZWN0PgogIDxwYXRoIGQ9Ik0xNiA4di0yYTIgMiAwIDAgMCAtMiAtMmgtOGEyIDIgMCAwIDAgLTIgMnY4YTIgMiAwIDAgMCAyIDJoMiI+PC9wYXRoPgo8L3N2Zz4=){.icon
.icon-tabler .icon-tabler-copy}
:::
:::

If you uncomment [`ld.add_action(spawn_turtlebot_cmd)`{.docutils
.literal .notranslate}]{.pre} in [`empty_world.launch.py`{.docutils
.literal .notranslate}]{.pre} and run the launch file, you'll notice
errors related to unrecognized plugins. These are coming from the model
SDFormat file, which we will modify next.
 
 
## Modify the model 

We will be using the [`waffle`{.docutils .literal .notranslate}]{.pre}
robot for this tutorial, so we'll edit the file
[[`turtlebot3_gazebo/models/turtlebot3_waffle/model.sdf`{.docutils
.literal
.notranslate}]{.pre}](https://github.com/ROBOTIS-GIT/turtlebot3_simulations/blob/d16cdbe7ecd601ccad48f87f77b6d89079ec5ac1/turtlebot3_gazebo/models/turtlebot3_waffle/model.sdf){.reference
.external}. The changes we need to make are mostly related to plugins
and their parameters. You can reference the Waffle [model SDF file
before editing
here](https://github.com/ROBOTIS-GIT/turtlebot3_simulations/blob/d16cdbe7ecd601ccad48f87f77b6d89079ec5ac1/turtlebot3_gazebo/models/turtlebot3_waffle/model.sdf){.reference
.external}, and [after editing
here](https://github.com/azeey/turtlebot3_simulations/blob/new_gazebo/turtlebot3_gazebo/models/turtlebot3_waffle/model.sdf){.reference
.external}. The following is a list of all the plugins in the original
model. For each plugin, we will either remove the plugin if it's no
longer necessary, or use the equivalent plugin from the new Gazebo. You
can use the Feature comparison page
([Fortress](https://gazebosim.org/docs/fortress/comparison){.reference
.external},
[Harmonic](https://gazebosim.org/docs/harmonic/comparison){.reference
.external}) to find out of a Gazebo Classic feature (e.g. a Sensor type)
is available in Gazebo. If an equivalent plugin is used, we will update
the SDF parameters of the plugin to match the parameters of the new
plugin. See the list of systems
([Fortress](https://gazebosim.org/api/gazebo/6/namespaceignition_1_1gazebo_1_1systems.html){.reference
.external},
[Harmonic](https://gazebosim.org/api/sim/8/namespacegz_1_1sim_1_1systems.html){.reference
.external}) to find equivalent plugins and their parameters. 

### libgazebo_ros_imu_sensor.so 

> <div>
>
> [Plugin in the original
> model](https://github.com/ROBOTIS-GIT/turtlebot3_simulations/blob/d16cdbe7ecd601ccad48f87f77b6d89079ec5ac1/turtlebot3_gazebo/models/turtlebot3_waffle/model.sdf#L89-L94){.reference
> .external}
>
> </div>

This plugin can be removed since there is a generic IMU plugin that
handles all IMU sensors. We will add this to the world later. We will
set the [`<topic>`{.docutils .literal .notranslate}]{.pre} tag inside
[`<sensor>`{.docutils .literal .notranslate}]{.pre} to a short topic
name to make it easier when creating a ROS bridge later. The entire
[`<sensor>`{.docutils .literal .notranslate}]{.pre} tag should now look
like:

 
```xml
<link name="imu_link">
  <sensor name="tb3_imu" type="imu">
    <always_on>true</always_on>
    <update_rate>200</update_rate>
    <topic>imu</topic>
    <imu>
      ... <!-- all the content of <imu> -->
    </imu>
  </sensor>
</link>
``` 

### libgazebo_ros_ray_sensor.so 

> <div>
>
> [Plugin in the original
> model](https://github.com/ROBOTIS-GIT/turtlebot3_simulations//blob/d16cdbe7ecd601ccad48f87f77b6d89079ec5ac1/turtlebot3_gazebo/models/turtlebot3_waffle/model.sdf#L157-L164){.reference
> .external}
>
> </div>

Similar to the IMU, we will use a generic plugin loaded into the world
for handling all rendering sensors, which includes Lidar sensors.
Currently the [`ray`{.docutils .literal .notranslate}]{.pre} sensor
type, which meant to use the physics engine for generating the sensor
data, is not supported in the new Gazebo, we will need to update it to
[`gpu_lidar`{.docutils .literal .notranslate}]{.pre}. We'll also need to
change the [`<ray>`{.docutils .literal .notranslate}]{.pre} tag inside
[`<sensor>`{.docutils .literal .notranslate}]{.pre} to
[`<lidar>`{.docutils .literal .notranslate}]{.pre}. The
[`frame_name`{.docutils .literal .notranslate}]{.pre} parameter of the
plugin will be handled by setting the [`gz_frame_id`{.docutils .literal
.notranslate}]{.pre} parameter in [`<sensor>`{.docutils .literal
.notranslate}]{.pre}. Lastly, we'll set the [`<topic>`{.docutils
.literal .notranslate}]{.pre} parameter similar to the IMU sensor. The
final [`<sensor>`{.docutils .literal .notranslate}]{.pre} tag for the
Lidar should look like:

 
```xml
<sensor name="hls_lfcd_lds" type="gpu_lidar">
  <always_on>true</always_on>
  <visualize>true</visualize>
  <pose>-0.064 0 0.121 0 0 0</pose>
  <update_rate>5</update_rate>
  <topic>scan</topic>
  <gz_frame_id>base_scan</gz_frame_id>
  <lidar>
    ... <!-- same content as <ray> in the original -->
  </lidar>
</sensor>
``` 
 
### libgazebo_ros_camera.so 

> <div>
>
> [Plugin in the original
> model](https://github.com/ROBOTIS-GIT/turtlebot3_simulations//blob/d16cdbe7ecd601ccad48f87f77b6d89079ec5ac1/turtlebot3_gazebo/models/turtlebot3_waffle/model.sdf#L393-L402){.reference
> .external}
>
> </div>

The Camera sensor will also use a generic plugin that handles all
rendering sensors loaded into the world. In the SDF file, we will set
the [`<topic>`{.docutils .literal .notranslate}]{.pre} tag inside
[`<sensor>`{.docutils .literal .notranslate}]{.pre}, and the
[`<camera_info_topic>`{.docutils .literal .notranslate}]{.pre} inside
[`<camera>`{.docutils .literal .notranslate}]{.pre}, both of which will
be used in the ROS bridge later. We will also set the
[`<gz_frame_id>`{.docutils .literal .notranslate}]{.pre} since the
default frame id used by the generic plugin in the new Gazebo is
different from the default used by [`libgazebo_ros_camera`{.docutils
.literal .notranslate}]{.pre} in Gazebo Classic. The final
[`<sensor>`{.docutils .literal .notranslate}]{.pre} tag should look
like:

 
```xml
<sensor name="camera" type="camera">
  <always_on>true</always_on>
  <visualize>true</visualize>
  <update_rate>30</update_rate>
  <topic>camera/image_raw</topic>
  <gz_frame_id>camera_rgb_frame</gz_frame_id>
  <camera name="intel_realsense_r200">
    <camera_info_topic>camera/camera_info</camera_info_topic>
    ... <!-- all the content of <camera> from the original -->
  </camera>
</sensor>
```
  
### libgazebo_ros_diff_drive.so 

> <div>
>
> [Plugin in the original
> model](https://github.com/ROBOTIS-GIT/turtlebot3_simulations//blob/d16cdbe7ecd601ccad48f87f77b6d89079ec5ac1/turtlebot3_gazebo/models/turtlebot3_waffle/model.sdf#L476-L507){.reference
> .external}
>
> </div>

Since this is a model specific plugin, we will replace it with the
[`DiffDrive`{.docutils .literal .notranslate}]{.pre} plugin. We will
match the parameters of [`libgazebo_ros_diff_drive`{.docutils .literal
.notranslate}]{.pre} as much as possible, but exact match may not be
possible. For example, the original plugin has a
[`max_wheel_acceleration`{.docutils .literal .notranslate}]{.pre}, but
[`gz-sim-diff-drive-system`{.docutils .literal .notranslate}]{.pre} has
[`max_linear_acceleration`{.docutils .literal .notranslate}]{.pre}
instead, which are not equivalent; the latter is a limit on the whole
vehicle's linear acceleration. We can approximate the value by
multiplying the wheel acceleration limit by the radius of the wheel.
Refer to the [DiffDrive class
reference](https://gazebosim.org/api/gazebo/6/classignition_1_1gazebo_1_1systems_1_1DiffDrive.html){.reference
.external} for details on each parameter. Here's the full
[`<plugin>`{.docutils .literal .notranslate}]{.pre} tag with comments
describing the mapping from the original plugin.

 
```xml
<plugin filename="gz-sim-diff-drive-system" name="gz::sim::systems::DiffDrive">
  <!-- Remove <ros> tag. -->

  <!-- wheels -->
  <left_joint>wheel_left_joint</left_joint>
  <right_joint>wheel_right_joint</right_joint>

  <!-- kinematics -->
  <wheel_separation>0.287</wheel_separation>
  <wheel_radius>0.033</wheel_radius> <!-- computed from <wheel_diameter> in the original plugin-->

  <!-- limits -->
  <max_linear_acceleration>0.033</max_linear_acceleration> <!-- computed from <max_linear_acceleration> in the original plugin-->

  <topic>cmd_vel</topic> <!-- from <commant_topic> -->

  <odom_topic>odom</odom_topic> <!-- from <odometry_topic> -->
  <frame_id>odom</frame_id> <!-- from <odometry_frame> -->
  <child_frame_id>base_footprint</child_frame_id> <!-- from <robot_base_frame> -->
  <odom_publisher_frequency>30</odom_publisher_frequency> <!-- from <update_rate>-->

  <tf_topic>/tf</tf_topic> <!-- Short topic name for tf output -->

</plugin>
```
 

The [`<wheel_torque>`{.docutils .literal .notranslate}]{.pre} parameter
can be realized by setting effort limits on each [`<joint>`{.docutils
.literal .notranslate}]{.pre}. For example:

 
```xml
<joint name="wheel_right_joint" type="revolute">
  <parent>base_link</parent>
  <child>wheel_right_link</child>
  <pose>0.0 -0.144 0.023 -1.57 0 0</pose>
  <axis>
    <xyz>0 0 1</xyz>
    <limit>
      <effort>20</effort> <!-- from <wheel_torque> in libgazebo_ros_diff_drive.so.-->
    </limit>
  </axis>
</joint>
```
 
 
### libgazebo_ros_joint_state_publisher.so 

> <div>
>
> [Plugin in the original
> model](https://github.com/ROBOTIS-GIT/turtlebot3_simulations//blob/d16cdbe7ecd601ccad48f87f77b6d89079ec5ac1/turtlebot3_gazebo/models/turtlebot3_waffle/model.sdf#L509-L517){.reference
> .external}
>
> </div>

We will replace this plugin as well with
[[`JointStatePublisher`{.docutils .literal
.notranslate}]{.pre}](https://gazebosim.org/api/gazebo/6/classignition_1_1gazebo_1_1systems_1_1JointStatePublisher.html){.reference
.external}. The parameters are mostly similar, however, the
[`<update_rate>`{.docutils .literal .notranslate}]{.pre} parameter is
not supported. Here's the full [`<plugin>`{.docutils .literal
.notranslate}]{.pre} tag with comments describing the mapping from the
original plugin.

 
```xml
<plugin filename="gz-sim-joint-state-publisher-system"
  name="gz::sim::systems::JointStatePublisher">
  <topic>joint_states</topic> <!--from <ros><remapping> -->
  <joint_name>wheel_left_joint</joint_name>
  <joint_name>wheel_right_joint</joint_name>
</plugin>
```
 

::: {#world-plugins .section}
### World plugins[\#](https://gazebosim.org/docs/harmonic/migrating_gazebo_classic_ros2_packages/#world-plugins "Link to this heading"){.headerlink}

As mentioned earlier, sensors are handled by generic world level
plugins.

::: warning
By default, if there are no world plugins specified, Gazebo adds the
Physics, SceneBroadcaster, and UserCommands plugins. However, if we
specify any plugins at all, Gazebo will assume we want to override the
defaults, so will not add any default plugins.
:::

Therefore, we have to add the additional plugins for IMU and Lidar
sensors as well as the ones that would have been added by default. We
will once again edit
[`turtlebot3_gazebo/worlds/empty_world.world`{.docutils .literal
.notranslate}]{.pre} add the following right after [`<world`{.docutils
.literal .notranslate}]{.pre}` `{.docutils .literal
.notranslate}[`name="default">`{.docutils .literal .notranslate}]{.pre}.
 
```xml
<plugin
  filename="gz-sim-physics-system"
  name="gz::sim::systems::Physics">
</plugin>
<plugin
  filename="gz-sim-user-commands-system"
  name="gz::sim::systems::UserCommands">
</plugin>
<plugin
  filename="gz-sim-scene-broadcaster-system"
  name="gz::sim::systems::SceneBroadcaster">
</plugin>
<plugin
  filename="gz-sim-sensors-system"
  name="gz::sim::systems::Sensors">
  <render_engine>ogre2</render_engine>
</plugin>
<plugin
  filename="gz-sim-imu-system"
  name="gz::sim::systems::Imu">
</plugin>
```

## Bridge ROS topics[\#](#bridge-ros-topics)

In Gazebo Classic, communication with ROS is enabled by plugins in
[`gazebo_ros_pkgs`{.docutils .literal .notranslate}]{.pre} that directly
interface with the simulator. In contrast, in the new Gazebo,
communication with ROS is mainly done through topic bridges provided by
[`ros_gz`{.docutils .literal .notranslate}]{.pre}. The bridge node is a
generic node that bridges topics between [`gz-transport`{.docutils
.literal .notranslate}]{.pre} and ROS 2.

To create the bridge, we'll use a [`yaml`{.docutils .literal
.notranslate}]{.pre} file that contains the topic names and their
mappings. We'll add a new directory [`params`{.docutils .literal
.notranslate}]{.pre} in [`turtlebot3_gazebo`{.docutils .literal
.notranslate}]{.pre} and create
[`turtlebot3_waffle_bridge.yaml`{.docutils .literal .notranslate}]{.pre}
with the following content:
 
```yaml
# gz topic published by the simulator core
- ros_topic_name: "clock"
  gz_topic_name: "clock"
  ros_type_name: "rosgraph_msgs/msg/Clock"
  gz_type_name: "gz.msgs.Clock"
  direction: GZ_TO_ROS

# gz topic published by JointState plugin
- ros_topic_name: "joint_states"
  gz_topic_name: "joint_states"
  ros_type_name: "sensor_msgs/msg/JointState"
  gz_type_name: "gz.msgs.Model"
  direction: GZ_TO_ROS

# gz topic published by DiffDrive plugin
- ros_topic_name: "odom"
  gz_topic_name: "odom"
  ros_type_name: "nav_msgs/msg/Odometry"
  gz_type_name: "gz.msgs.Odometry"
  direction: GZ_TO_ROS

# gz topic published by DiffDrive plugin
- ros_topic_name: "tf"
  gz_topic_name: "tf"
  ros_type_name: "tf2_msgs/msg/TFMessage"
  gz_type_name: "gz.msgs.Pose_V"
  direction: GZ_TO_ROS

# gz topic subscribed to by DiffDrive plugin
- ros_topic_name: "cmd_vel"
  gz_topic_name: "cmd_vel"
  ros_type_name: "geometry_msgs/msg/Twist"
  gz_type_name: "gz.msgs.Twist"
  direction: ROS_TO_GZ

# gz topic published by IMU plugin
- ros_topic_name: "imu"
  gz_topic_name: "imu"
  ros_type_name: "sensor_msgs/msg/Imu"
  gz_type_name: "gz.msgs.IMU"
  direction: GZ_TO_ROS

# gz topic published by Sensors plugin
- ros_topic_name: "scan"
  gz_topic_name: "scan"
  ros_type_name: "sensor_msgs/msg/LaserScan"
  gz_type_name: "gz.msgs.LaserScan"
  direction: GZ_TO_ROS

# gz topic published by Sensors plugin (Camera)
- ros_topic_name: "camera/camera_info"
  gz_topic_name: "camera/camera_info"
  ros_type_name: "sensor_msgs/msg/CameraInfo"
  gz_type_name: "gz.msgs.CameraInfo"
  direction: GZ_TO_ROS
```
 

[The completed yaml file can be found
here.](https://github.com/azeey/turtlebot3_simulations/blob/new_gazebo/turtlebot3_gazebo/params/turtlebot3_waffle_bridge.yaml){.reference
.external} Each entry in the yaml file has a ROS topic name, a Gazebo
topic name, a ROS data/message type, and a direction which indicates
which way messages flow. We will need to update the
[[`CMakeLists.txt`{.docutils .literal
.notranslate}]{.pre}](https://github.com/azeey/turtlebot3_simulations/blob/ccb2385fd478e79c898d290d80fa41f35b1bbb83/turtlebot3_gazebo/CMakeLists.txt#L72){.reference
.external} file to install the new [`params`{.docutils .literal
.notranslate}]{.pre} directory we created. The CMake
[`install`{.docutils .literal .notranslate}]{.pre} command should look
like

::: {.highlight-cmake .notranslate}
::: highlight
``` {#codecell24 tabindex="-1"}
install(DIRECTORY launch models params rviz urdf worlds
  DESTINATION share/${PROJECT_NAME}/
)
```
 

Finally, we will edit
[[`turtlebot3_gazebo/launch/spawn_turtlebot3.launch.py`{.docutils
.literal
.notranslate}]{.pre}](https://github.com/ROBOTIS-GIT/turtlebot3_simulations//blob/d16cdbe7ecd601ccad48f87f77b6d89079ec5ac1/turtlebot3_gazebo/launch/spawn_turtlebot3.launch.py) , to create the bridge node:
 
```py
bridge_params = os.path.join(
    get_package_share_directory('turtlebot3_gazebo'),
    'params',
    'turtlebot3_waffle_bridge.yaml'
)

start_gazebo_ros_bridge_cmd = Node(
    package='ros_gz_bridge',
    executable='parameter_bridge',
    arguments=[
        '--ros-args',
        '-p',
        f'config_file:={bridge_params}',
    ],
    output='screen',
)
```
 

You might have noticed that in the bridge parameters, we did not include
the [`camera/image_raw`{.docutils .literal .notranslate}]{.pre} topic.
While it is possible to bridge the image topic in a similar manner as
all the other topics, we will make use of a specialized bridge node,
[[`ros_gz_image`{.docutils .literal
.notranslate}]{.pre}](https://github.com/gazebosim/ros_gz/tree/ros2/ros_gz_image){.reference
.external}, which provides a much more efficient bridge for image
topics. We'll add the following snippet to
[`turtlebot3_gazebo/launch/spawn_turtlebot3.launch.py`] 

 
```py
start_gazebo_ros_image_bridge_cmd = Node(
    package='ros_gz_image',
    executable='image_bridge',
    arguments=['/camera/image_raw'],
    output='screen',
)
```
 
Finally, we will add all the new actions to the list of
[`LaunchDescription`{.docutils .literal .notranslate}]{.pre}s returned
by the [`generate_launch_description`{.docutils .literal
.notranslate}] 

```sh
# ...

# Add the action to `ld` toward the end of the file

ld.add_action(start_gazebo_ros_bridge_cmd)
ld.add_action(start_gazebo_ros_image_bridge_cmd)
```
 
You can find the [Gazebo Classic [`spawn_turtlebot3.launch.py`{.docutils
.literal .notranslate}]{.pre} file
here](https://github.com/ROBOTIS-GIT/turtlebot3_simulations/blob/d16cdbe7ecd601ccad48f87f77b6d89079ec5ac1/turtlebot3_gazebo/launch/spawn_turtlebot3.launch.py){.reference
.external}, and the [updated Gazebo
[`spawn_turtlebot3.launch.py`{.docutils .literal .notranslate}]{.pre}
file
here](https://github.com/azeey/turtlebot3_simulations/blob/new_gazebo/turtlebot3_gazebo/launch/spawn_turtlebot3.launch.py) 

We are now ready to launch the empty world which spawns the waffle robot
and sets up the bridge so that we can communicate with it from ROS 2.

 
```sh
export TURTLEBOT3_MODEL=waffle
ros2 launch turtlebot3_gazebo empty_world.launch.py
```



Here's a screenshot of Turtlebot3 running in Gazebo obtained by
launching [`empty_world.launch.py`{.docutils .literal
.notranslate}]{.pre}. The Lidar visualization is enabled by adding the
"Visualize Lidar" GUI plugin (see
[tutorial](https://gazebosim.org/docs/fortress/gui){.reference
.external} on how to add GUI plugins).

![Screenshot of turtlebot 3 running in
Gazebo](./migrating-ros2-packages-from-gazebo-classic_files/gazebo_turtlebot3.png)

It is also now possible to do the
[SLAM](https://emanual.robotis.com/docs/en/platform/turtlebot3/slam_simulation/){.reference
.external} and
[Navigation](https://emanual.robotis.com/docs/en/platform/turtlebot3/nav_simulation/){.reference
.external} tutorials from the Turtlebot3 manual (make sure to select the
Humble tab). However, it requires updating
[`turtlebot3_world.world`{.docutils .literal .notranslate}]{.pre} and
[`turtlebot3_world.launch.py`{.docutils .literal .notranslate}]{.pre}
files according what we've discussed in this tutorial. For reference,
those files have also been migrated in [this
fork](https://github.com/azeey/turtlebot3_simulations/tree/new_gazebo) 
 
 
## Migrating other files in turtlebot3_gazebo[\#](https://gazebosim.org/docs/harmonic/migrating_gazebo_classic_ros2_packages/#migrating-other-files-in-turtlebot3-gazebo "Link to this heading"){.headerlink}

This tutorial does not cover all aspects of migrating models and launch
files from Gazebo classic. Please see the [[Gazebo Classic
Migration]{.doc .std
.std-doc}](https://gazebosim.org/docs/harmonic/gazebo_classic_migration/){.reference
.internal} document for more resources that help with migrating other
aspects, such as Gazebo Classic plugins, materials and textures.
