HBBA - Hybrid Behavior-Based Architecture
=========================================

This is the (in progress) implementation of HBBA, the robot control architecture
built at 3IT-IntRoLab (Université de Sherbrooke).
It has been built with ROS and our robots (IRL-1/TR and /AZ3 variants) in mind,
but has been designed to be compatible with various platforms.

Main concepts
-------------

HBBA, for Hybrid Behavior-Based Architecture, is based on a few concepts:

 - HBBA is composed of three layers: the Organization layer, where Motivation
   modules are, the Coordination layer, containing the Intention Workspace (IW)
   and its associated modules, and the Behavioral layer, containing Perception
   and Behavior modules.
 - Motivation modules are independent, high-level modules that manage goals for
   the robot. Goals are described as Desires, which can recommend or inhibit 
   actions such as going somewhere or expressing speech, but also information
   processing tasks such as object recognition and SLAM.
 - Perception modules take raw sensor information and transform it into
   percepts for the Behavior modules. A typical example would be face detection,
   where raw camera images are processed to produce the location of a person's
   face, if present.
 - Behavior modules are independent processes that take percepts and generate
   commands for the actuators of the robot. Access to the actuators is
   arbitrated, usually in a priority-based subsumption scheme.
 - The role of the IW is to contain all current Desires of the robot.
   One of its modules, the IW Translator, converts the active set of Desires
   into the Intention of the robot. Desires are normally robot-independent, 
   which means they describe *what* the robot should do, but not *how*.
 - The IW Translator relies on a database of Strategies to perform the
   conversion. Desires are categorized by class, and each strategy describe a
   way to fulfill a single class of Desires.
 - Strategies are characterized by Utility and costs in resources. A Strategy
   with higher Utility usually requires more resources, and vice versa. The role
   of the IW Translator is to fulfill as many Desires as possible while
   respecting the limited resources of the robot.
 - The Intention of the robot is rendered by configuring the data flow between
   all modules of the architecture through filters.
   For instance, to enable SLAM, a filter between a data source such as a laser
   range finder and the SLAM Perception module has to be opened. Strategies do
   not have to be all on/all off: they can also specify reduced data rate to
   reduce resource consumption while still providing some Utility to the Desire
   class.
 - Only one Desire per class can be fulfilled at the same time. To avoid
   conflicts, Desires are parametrized by Intensity, and priority is given to
   the most intense Desire per class.

There are three main parts to consider when running HBBA on a robot with ROS:

 - The Hardware Abstraction Layer (HAL): This consists in low-level sensor and
   control ROS nodes. 
   This will vary from robot to robot.
   It can also be a Gazebo simulation.
 - HBBA core nodes: This consists in the Intention Workspace (IW), IW Translator
   and script engine. This part relies on many dependencies, but can be 
   contained in a Docker virtual machine (see hbba_docker).
 - Higher-level motivation, perception and behavior nodes. These nodes can
   communicate with both the HAL and HBBA core nodes, and include the HBBA
   filters.

Thus, the typical steps necessary to start a robot with HBBA are:

 - Launch the HAL with a single launch file. At this point, the robot sensors
   and actuators are directly accessible through ROS.
 - Load the HBBA model into the ROS parameter server. This model describes the
   Strategies and robot resources to be used by the IW Translator. This model is
   usually automatically generated by hbba_synth.
 - Launch the HBBA core nodes. When they start, the core nodes read the model
   from the parameter server to initialize themselves.
 - Launch the higher-level nodes. A launch file for this is also usually
   generated by hbba_synth, as it also include the filters configuration.
 - Add static desires to the IW. This can be done by a launch file and/or python 
   script, both also generated by hbba_synth.

Obtaining and building HBBA
---------------------------

HBBA itself relies on many dependencies, most of them related to Google
or-tools, which is used by the IW Translator.
Building the whole project locally is recommended, but we also have a
Docker-based configuration that isolate some of the dependencies.

### Complete build on Ubuntu 18.04 and ROS Melodic (recommended)

Google or-tools has to be installed first to build the Intention Translator
(iw_translator).
Unfortunately, it is not currently offered as a Debian package, but can be built
from source or downloaded as a pre-built version for Ubuntu 18.04.
Also, see the "or_tools" subfolder of the iw_translator package for a dpkg
generation script that can then be used to install or-tools.
To use it, simply run:

  or_tools$ ./generate_dpkg.sh
  or_tools$ sudo dpkg -i or-tools_ubuntu-18.04_v7.1.6720.deb

This installs the needed librairies in /opt/or-tools.
The iw_translator package is configured to point to this location by default.

The whole distribution also requires those system dependencies:

 - bison
 - flex
 - libv8-dev

To build the distribution, do not forget to fetch the hbba_base submodule like
this (from the root directory of this repository):

$ git submodule init; git submodule update

Then, the whole system should build from a single catkin_make.
If you encounter errors related to a package such as "hbba_synth" not being
found, this usually means that the rospkg cache needs to be updated.
Re-sourcing the setup.bash script and re-running catkin_make usually solves
this. 

### Docker-based build 

To simplify the building process, a Docker container based on Ubuntu 14.04 and
ROS Indigo has been made.
Therefore, a typical HBBA configuration would have the ROS master
daemon, low-level sensor and actuator driver nodes and higher-level nodes 
running directly in the working environment, while the HBBA core nodes stay
in a Docker container on that same PC.

Docker is a virtual machine management tool for distributed applications, where
each component (often a single process such as a database or web service) is
isolated in its own container. On Linux, Docker does not rely on full system
virtualization, but instead perform OS-level resource isolation with cgroups and
namespaces. On other systems, VirtualBox can be used.

Installing Docker on Ubuntu 14.04 is as a simple as

    $ sudo apt-get install docker.io

The Docker daemon is automatically configured and accessible with the 'docker'
command line tool. Note that you have to add your account to the 'docker'
group if you want to use docker outside of sudo:

    $ sudo addgroup <user> docker

After a logout/login cycle, you should have access to docker. You can use 

    $ docker ps
    
To list the currently running docker containers. If you get an error regarding
permissions, you probably did something wrong when adding your user to the
docker gorup.

This repository is meant to be part of a ROS workspace, and contains 
sub-modules in addition to normal packages:

 - hbba_base: These are base HBBA ROS packages with no external dependencies. It
   contains message definitions, the HBBA configuration generator (hbba_synth)
   and a few command line tools to interact with the core nodes.
 - hbba_docker: The Docker configuration for the full HBBA distribution with 
   core nodes. While it is not a traditional ROS package, it comes with a script
   to build the Docker container on demand.

Assuming your workspace is in catkin_ws, these commands will clone the
repository, fetch its submodules, and build all the pacakges, except for the
Docker container:

    $ cd ~/catkin_ws/src
    $ git clone [url to this repository] hbba 
    $ cd hbba
    $ git submodule init
    $ git submodule update
    $ cd ~/catkin_ws
    $ catkin_make

If this repository is part of another, larger distribution, simply run the
submodule commands from the hbba subfolder.
Periodically, the submodules can be updated with:

    $ cd ~/catkin_ws/src/hbba
    $ git submodule foreach git pull
    
To fetch a pre-built image of the Docker container, you can use:

    $ docker pull francoisferland/hbba

You can also build the Docker container using the provided build_docker.sh script:

    $ rosrun hbba_docker build_docker.sh

Please note that the build process might take several minutes.
You might also encounter an error with CMake saying 'file is busy' at one
point in the process.
This is related to bug with the default Docker filesystem driver, which does not
correctly sync its files.
For the moment, you can simply re-run the build command until the error
disappears.

Turtlebot demo
--------------

The package includes a Turtlebot-based demo configuration to test if everything
has been installed correctly.
It relies on a Gazebo-based simulation of a Turtlebot3 and requires its full
software suite.
If you do not already have it, you can install it like this:

    $ sudo apt-get install ros-melodic-turtlebot3 ros-melodic-turtlebot3-simulations

NOTE: As of 2019/09/05, the turtlebot3_navigation package as a bug in the binary
version related to TF frame names.
You can instead clone the turtlebot3 and turtlebot3_simulations repos from
GitHub in your workspace.
They also require the installation of the whole navigation stack and gmapping :

    $ sudo apt-get install ros-melodic-navigation ros-melodic-gmapping

Then, if everything built correctly, you can start the whole system:

    $ export TURTLEBOT3_MODEL=waffle
    $ roslaunch turtlebot_hbba_cfg turtlebot_gazebo.launch
    $ roslaunch turtlebot_hbba_cfg turtlebot_nav_slam.launch

The first line sets the TURTLEBOT3_MODEL environment variable that is required
by other Turtlebot3 nodes.
It then launches the simulator along with basic nodes, which corresponds to the
HAL of the robot.
Finally, the last line starts the HBBA config for navigation with SLAM.
This takes care of three things:
 
 - Load the model in the parameter server;
 - Launch higher-level nodes, which includes a navigation behavior;
 - Add initial Desires when the core nodes are ready.

The idea of splitting in two steps is that it allows to restart the HBBA
runtime (step two) without restarting the simulator or HAL of the real
robot.

If everything is running correctly, you should see a Gazebo window with the
robot and a simple environment.
At this point, you can use iw_console to manually add a Desire to the IW:

    $ rosrun iw_tools iw_console
    > add GoTo 1 1 {frame_id: 'map', x: 3.0, y: 3.0, t: 0.0}
    > quit

The 'add' line creates a Desire of the GoTo class, with an intensity of 1 and a 
utility requirement of 1.
The remaining part of the line is interpreted as JSON and sent to the Desire
activation script. Here, the structure indicates that the goal is in the 'map' 
TF frame, at a position of (x,y) = (3.0, 3.0), with an orientation (theta angle)
of 0.0 radians relative to the X axis.

If everything worked correctly, the robot should slowly move to the requested
location.

Furthermore, the configuration includes a motivation module that converts 
geometry_msgs/PoseStamped messages into GoTo desires, and remove them
automatically when the robot reaches its goal.
This is managed by the goto_generator node, which accepts new goals on the
/goto_generator/goal topic, which can be generated with RViz.
A sample configuration for RViz is also available in the turtlebot_hbba_cfg/cfg
directory that is already configured for this.
