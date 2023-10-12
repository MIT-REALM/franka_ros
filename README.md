**NOTE: this is a template repository. Do not clone it; instead, click "Use this template" on GitHub**

# Dockerized Franka ROS interface

This repository allows you to run [`libfranka`](https://frankaemika.github.io/docs/libfranks.html) and [`franka_ros`](https://frankaemika.github.io/docs/franka_ros.html) in a Docker environment (complete with ROS Noetic, GUI access, and networking to the robot with the `FCI_IP` environment variable).

To build:
```bash
docker compose build
```

To run
```bash
docker compose up
# Open a new terminal window and run:
docker attach <your-repo-name>-main-1
```
where `<your-repo-name>` is the name that you choose for the new repository (i.e., the repository that uses this template).

Now you should have access to the full Franka ROS interface documented [`here`](https://frankaemika.github.io/docs/franka_ros.html).

For example, you can test the connection to the robot as follows:

1. Go to the [Desk webapp](https://172.16.0.2/desk/) and login.
2. Unlock the robot using the sidebar on the right of the screen.
3. Open the drop-down in the top right by clicking on the IP address, then click "Activate FCI" to enable the Franka Control Interface.
4. Run the docker container using `docker compose run main`
5. Run the command `roslaunch franka_example_controllers joint_impedance_example_controller.launch   robot_ip:=$FCI_IP load_gripper:=true robot:=fr3`

The robot should move in a circle, and an RViz window should open showing a matching 3D model of the robot.

## Controlling the robot via ROS

In theory, the `franka_ros` Docker image provides a full installation of all ROS packages needed for full control of the arm. Unfortunately, "full control" means writing a C++ controller to run in real time. To support easier control of the arm, we provide a `docker compose` option that will launch a cartesian impedance controller that will track an equilibrium pose published as a ROS topic.

To run this controller:

1. Run `docker compose up`. This will launch 3 Docker containers:
    - `franka_ros-roscore-1`: runs `roscore`
    - `franka_ros-cartesian_impedance_control-1`: runs the impedance controller, RViz (for visualizing the state of the robot), and a GUI utility for modifying the impedance control stiffnesses of the robot.
    - `franka_ros-main-1`: runs a bash shell (**this is where you should run your code**).
2. In a new terminal tab, run `docker attach franka_ros-main-1`. This will give you a bash shell that can talk to the robot via ROS. From here, you can publish `geometry_msgs/PoseStamped` messages to the `/cartesian_impedance_example_controller/equilibrium_pose` topic to move the arm.

When you attach to the `franka_ros-main-1` container, you should see that any files in the `franka_realm` folder have been mounted to the container. This folder is a ROS package, so any scripts you add there will be accessible from inside the container.

## Adding new components

By default, the `docker-compose.yml` file launches 3 containers (called "services"):

- `roscore`: runs roscore.
- `cartesian_impedance_control`: runs the impedance controller to track an end effector pose.
- `main`: to provide a bash terminal for the user.

You can add additional services by adding them to the `docker-compose.yml` file:

```yml
services:
  
  ...

  my_new_service:
    image: franka_ros:latest  # which docker image to use
    build: .docker  # where that dockerfile is stored
    environment:
      - DISPLAY=${DISPLAY}  # allows GUI access
    volumes:
      - /tmp/.X11-unix:/tmp/.X11-unix  # allows GUI access
      - ./realm:/catkin_ws/src/realm  # allows access to the ROS package in the realm directory
    network_mode: host  # allows access to the host's network (and the robot)
    privileged: true  # needed for GUI + USB access
    stdin_open: true # allows you to attach to a shell in this container
    tty: true        # allows you to attach to a shell in this container
    depends_on:  # don't start until these have started
      - roscore
      - cartesian_impedance_control
    command: bash  # the command to run when starting this container.
```

The main things to edit here are changing which volumes get mounted (e.g. if you want to
access different files) and the command to run when started (e.g. if you want to run some
launch file from a package you've mounted). If you want to install additional dependencies in the docker image, you can modify `.docker/Dockerfile`.
