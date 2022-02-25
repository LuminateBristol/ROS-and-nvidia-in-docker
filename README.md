# Using ROS Melodic through a docker container

Note 1 - I am no expert in Docker or ROS, this is purely a reference guide based on my experiences in combining the two.

Note 2 - This method works for any version of ROS and any version of host machine as long as the docker version that is installed, contains the relevent builds for the ROS version that you need.

## A Little More About Docker and Containers
Containers allow us to create a virtualisation of an operating system which we can run, from a linux system and use to run software on whilst keeping it seperate from our main linux OS. This is useful for testing software packages or for running software that needs a different OS to the one that is currently being ran on the host machine.

Docker helps make containers easy to use by providing a set of commands and various other useful scripts to make setting up containers quick and easy.

Docker works through images and containers. First an image is created, this is the basis on which containers will work. Images can be created using dockerfiles which provide information on the build of the image. For example the dockerfile can specify that the image build requires Nvidia GPU acceleration packages.

Containers are then created based on the image and these are our workspaces. They take the attributes of an image and allow us to work in build through a command line interface (CLI). We can start and stop containers to carry on from our previous work OR we can commit the container to a new image to create a more permenant saved snapshot. Comitting to a new image is recommended after major changes but remember that to then work on that version again, we will need to create a new container based on the new image.

When we 'run' an image, a new container is created. When we 'start' a container, we continue working on an existing container.

## Setting up Docker Containers with ROS

### STEP 1 - SETUP

Install docker (latest version) - https://docs.docker.com/get-docker/

### STEP 2 - INSTALL ROS

Setting up a simple ROS Melodic Docker Image is simple because Docker has the build files ready to use with a single line of code. This single line creates an image that is pre-setup with ROS Melodic and the relevent Ubuntu version (18.04). This can be done for most versions of ROS.

In the terminal:
```
$ sudo docker pull ros:melodic
```
Check the version is correct and see the Image ID
```
$ sudo docker images
```

### STEP 3 - USING ROS

To start using ROS we first need to create a container based on the ROS image that we just built. To do so we use the `run` command which creates a new container (always a new container with this command!) based on a specified image. We add the `-it` to ensure that Docker provides us with a CLI to type into. In our new CLI, start ros with roscore.

Note - `<mage ID>` or `<container ID>` can be replaced with `<image name>` or `<container name>` in all cases.


```
$ sudo docker run -it <image ID>
$ roscore
```

Now by ROs' very nature, to do anything useful we need to open a new terminal. Unfortunately simply opening a terminal like normal will create a new one on our host machine but not a new one within our container. We therefore need a couple of extra steps. 

First we find the container ID of the container that we just created with the 'run' command. Then we setup a new terminal using that container ID:
```
$ sudo docker ps
$ sudo docker exec -it <container ID> bash
```

Then as normally needed for ROS (note as with standard ROS, the .bashrc script can be updated to avoid typing this every time):
```
$ source /opt/ros/melodic/setup.bash
```

Repeat this process everytime you need a new terminal (slightly annoying but 
only one extra step over standard ROS use).

### STEP 4 - USING GRAPHICS
Whilst the above works for basic ROS use, if we want to use RViz, Gazebo etc then we need to use the NVIDIA drivers to allow this

First install NVIDIA using the following tutorial: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#setting-up-nvidia-container-toolkit

Then setup a new image with both NVIDIA and ROS using this tutorial: http://wiki.ros.org/action/login/docker/Tutorials/Hardware%20Acceleration#nvidia-docker2

(Note that the code stored in this repo is taken from this tutorial.)

Note that if the second tutorial does not work straight away, try adding the following before the 'build' command:
```
$ sudo rm -rf /tmp/.docker.xauth
```

You should now have a new image in your list of images.
```
$ sudo docker images
```

NOTE - you may get the error "Could not connect to any X display" when opening a new container with NVIDIA. Solve this by allowing the container access to xhost by running the following command prior to starting / opening the container:
```
xhost +local:docker
```


### STEP 5 - CREATING A CONTAINER IN THE NEW IMAGE

The run command is slightly more complex for this image. If you followed the tutorial then you will already have a container running within the image. To run a new container we have to use the run command from the tutorial:
```

$ sudo docker run -it \
     --env="DISPLAY=$DISPLAY" \
     --env="QT_X11_NO_MITSHM=1" \
     --volume="/tmp/.X11-unix:/tmp/.X11-unix:rw" \
     --env="XAUTHORITY=$XAUTH" \
     --volume="$XAUTH:$XAUTH" \
     --runtime=nvidia \
     my_melodic_image \
     bash
```
     
To make this easier we can add this to an executable bash file and run using './<file name.bash>' but remember to make that file executable using 'chmod a+x <file name.bash>'. (See tutorial for more).

And we are in! We've opened a container of the image which we can now used as a standard ubuntu command line (CLI) with ROS in the same way as described in Step 3.

Finally we want to update the .bashrc file to make sure the graphics work and to prevent us having to source ros every time. (Note you may need to install gedit too). In the command line for the newly ran container:

```
$ source /opt/ros/melodic/setup.bash
$ export XDG_RUNTIME_DIR=/some/directory/you/specify 
$ export RUNLEVEL=3
```

### STEP 6 - MAKING CHANGES

If we make any changes in the container there are two options to save the changes for future work.
1) Stop the container and then start it again

In a new host computer terminal:
```
$ sudo docker stop <container ID>
```
Or simply type 'exit' whilst in the container.

Start the container back up again and reattach:
```
$ sudo docker start <container ID>
$ sudo docker attach <container ID>
```

2) Commit the container to a new image (for more serious changes for example)
```
$ sudo docker commit <container> <new image name>
```

Then to run we use the same code as before but with the new image name to start a new container based on the new image.
```
$ sudo docker run -it \
     --env="DISPLAY=$DISPLAY" \
     --env="QT_X11_NO_MITSHM=1" \
     --volume="/tmp/.X11-unix:/tmp/.X11-unix:rw" \
     --env="XAUTHORITY=$XAUTH" \
     --volume="$XAUTH:$XAUTH" \
     --runtime=nvidia \
     <new image name> \
     bash
```

### SOME OTHER USEFUL COMMANDS:

Remove all stopped containers (be careful!)
```
$ sudo docker container prune
```

Remove image without and with force
```
$ sudo docker image rm <image ID>
$ sudo docker image rm --force <image ID>
```

Remove container without and with force
```
$ sudo docker container rm <container ID>
$ sudo docker container rm --force <container ID>
```

Rename a container
```
$ sudo docker rename <old container name> <new container name>
```

Name a docker container, add the following to the docker run command
```
--name <container name>
```

### EXTRA: USING ARDUINO IN DOCKER

Install arduino on Docker:

METHOD 1: OLD VERSION OF ARDUINO
```
$ sudo apt-get install arduino
$ arduino
```

Installing libraries:
Download from the command line and then rename to suitable name (if needed - Arduino IDE will
throw a name error if it's got punctuation etc)
```
$ sudo apt-get install wget
$ wget -P /home/downloads <library url (get from arduino website>
$ unzip <libdary zip file>
$ mv <old library name> <new library name>
```
Now open arduino, go to Sketch -> Add library -> navigate and upload


Note that this method is only available for the old versions of Arduino and so does not work well with newer scripts.

METHOD 2: ARDUINO-CLI:

Navigate to $PATH and download the arduino-cli
```
$ echo $PATH
$ cd <navigate to a directory in $PATH>
$ wget "<link for latest arduino-cli>"
$ tar -xf arduino_cli...........tar.gz
```
You should now be able to run arduino through the arduino-cli.

Find a folder for a new sketch and run the following test code:
```
$ arduino-cli board listall mkr
$ arduino-cli sketch new test_script
$ arduono-cli cat test_script.ino
$ gedit test_script/test_script.ino
# ADD CODE #
$ arduino-cli compile --fqbn <FQBN> test_script.ino
$ arduino-cli upload -p <Port Name> --fqbn <FQBN> test_script.ino
```
Note you can find `<FQBN>` and `<Port Name>` using `arduino-cli board list` . An example FQBN for an Arduino Uno is `arduino:avr:uno` and an example port is `/dev/ttyUSB0`

Note that in many cases, Arduino libraries will need installing. This can be done using the following. Note that the library name needs to be correct - try googling the library name if it doesn't recognise it first time.
```
$ arduino-cli lib install <library name>
```


