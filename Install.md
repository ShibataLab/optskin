# Finger Vision

ROS package to interact with finger vision sensors developed by [Prof. Yamaguchi](http://akihikoy.net/info/).

* [Resources](#resources)
* [Requirements](#software)
* [Installation](#installation)
  * [Baxter PC](#pc)
  * [Raspberry Pi](#rpi)
* [Usage](#usage)
  * [Standalone](#standalone)
  * [ROS and Baxter](#ros)

## Resources <a name="resources"></a>

* Official project page: [Link](http://akihikoy.net/notes/?project%2FFingerVision)
* Manufacturing process: [Link](http://akihikoy.net/notes/?project%2FFingerVision%2FManufacturing)
* Software installation instructions: [Link](http://akihikoy.net/notes/?project%2FFingerVision%2FSoftware)
* Testing the finger vision sensors: [Video1](https://youtu.be/rv4B4Isdm5w), [Video2](https://youtu.be/Yu2XWvzr3og).
* System overview used in CMU: [Link](http://akihikoy.net/notes/?text%2FBaxter-2).

## Requirements <a name="requirements"></a>

### Hardware <a name="hardware"></a>

* Baxter Research Robot with electric grippers
* Finger vision sensors x 4
* Raspberry Pi 3B x 2

### Software <a name="software"></a>

* Baxter PC with Ubuntu 14.04, ROS Indigo, OpenCV >= 2.4.13
* Raspberry Pi with Ubuntu Mate 16.04

## Installation <a name="installation"></a>

### Baxter PC <a name="pc"></a>

* Setup Baxter PC with Ubuntu 14.04, ROS Indigo and baxter sdk workspace
* Install following ros-indigo dependencies:
  ```
  $ sudo apt-get install ros-indigo-roscpp ros-indigo-std-msgs ros-indigo-std-srvs ros-indigo-pcl-ros ros-indigo-image-transport ros-indigo-camera-info-manager ros-indigo-rosmake 
  ```
* Build OpenCV 2.4.13 from source and install to .local folder:
  ```
  $ sudo apt-get -f install libv4l-0 libv4l-dev
  $ sudo add-apt-repository ppa:mc3man/trusty-media
  $ sudo apt-get update
  $ sudo apt-get dist-upgrade
  $ sudo apt-get install ffmpeg
  $ git clone --branch 2.4 --depth 1 https://github.com/Itseez/opencv.git
  $ cd opencv
  $ mkdir build
  $ cd build
  $ cmake -D CMAKE_BUILD_TYPE=RELEASE -D CMAKE_INSTALL_PREFIX=$HOME/.local ..
  $ make -j8
  $ make install
  ```
  If this process does not work, then it might be necessary to build ffmpeg from source. The instructions for doing this are available [here](https://junise.wordpress.com/2015/05/18/how-to-install-opencv-2-4-10-in-ubuntu-14-04-lts/).

* Download the project source from this github [repo](https://github.com/akihikoy/optskin):
  ```
  $ cd ros_ws/
  $ git clone https://github.com/akihikoy/optskin
  ```
* Compile the stand alone programs:
  ```
  $ cd optskin/standalone/
  $ g++ -g -Wall -O2 -o simple_blob_tracker4.out simple_blob_tracker4.cpp -I$HOME/.local/include -L$HOME/.local/lib -Wl,-rpath=$HOME/.local/lib -lopencv_core -lopencv_imgproc -lopencv_features2d -lopencv_highgui
  $ g++ -g -Wall -O2 -o obj_det_track3.out obj_det_track3.cpp -I$HOME/.local/include -L$HOME/.local/lib -Wl,-rpath=$HOME/.local/lib -lopencv_core -lopencv_video -lopencv_imgproc -lopencv_highgui
  $ g++ -g -Wall -O2 -o capture.out capture.cpp -I$HOME/.local/include -L$HOME/.local/lib -Wl,-rpath=$HOME/.local/lib -lopencv_core -lopencv_highgui
  ```
* Build lfd_vision ros package:
  ```
  $ cd optskin/lfd_vision/
  $ export ROS_PACKAGE_PATH=$ROS_PACKAGE_PATH:`pwd`
  $ rosmake
  ```
  Check the config files `config/usbcam2fslab*.yaml`. **Make sure to modify the urls for the `DevID` tag to the URL obtained from MJPG-Streamer which will be installed later.**

### Raspberry Pi <a name="rpi"></a>

The sensors when placed on Baxter need to be connected to a Raspberry Pi. The camera data streamed over http from the RPi. Following are the instructions to setup MJPG-Streamer:

* Download Ubuntu Mate 16.04 for RPi from the [official website](https://ubuntu-mate.org/raspberry-pi/). **IMPORTANT!: The latest version 16.04.2 does not work well and it is important to download Ubuntu Mate 16.04**.
* Prepare SD card of atleast 8 GB and install the Ubuntu image on SD card using these [instructions](https://ubuntu-mate.org/raspberry-pi/). First use `lsblk` and look for the device with matching size to check for the SD card device file name:
  ```
  $ lsblk
  ```
  The device ID will be of the form `/dev/sdx`. Use this ID to install image in SD card:
  ```
  $ sudo apt-get install gddrescue xz-utils
  $ unxz ubuntu-mate-16.04-desktop-armhf-raspberry-pi.img.xz
  $ sudo ddrescue -D --force ubuntu-mate-16.04-desktop-armhf-raspberry-pi.img /dev/sdx
  ```
* Insert SD card in RPi and power it up. Install Ubuntu Mate by following the instructions.
* Login to the RPi. Set the clock to the current time as it usually causes network errors.
* Install [MJPG-streamer](https://github.com/jacksonliam/mjpg-streamer) to stream the camera data over network:
  ```
  $ sudo apt-get install raspi-config cmake git libjpeg8-dev uvcdynctrl v4l-utils
  $ git clone https://github.com/jacksonliam/mjpg-streamer
  $ cd mjpg-streamer/mjpg-streamer-experimental/
  ```
  Comment out the following line in `CMakeLists.txt`
  ```
  use_subdirectory(plugins/input_opencv)
  ```
  Build the package
  ```
  $ make
  ```
* Setup startup script `startup.sh` to enable two connected finger vision sensors:
  ```
  #!/bin/bash -x
  cd Documents/mjpg-streamer/mjpg-streamer-experimental
  ids=""
  ./mjpg_streamer -i "./input_uvc.so -f -1 -r 320x240 -d /dev/video0 -n" -o "./output_http.so -w ./www -p 8080" &
  ids="$ids $!"
  ./mjpg_streamer -i "./input_uvc.so -f -1 -r 320x240 -d /dev/video1 -n" -o "./output_http.so -w ./www -p 8081" &
  ids="$ids $!"
  sleep 1
  echo "Press enter to quit"
  read s
  kill -SIGINT $ids
  sleep 2
  ```
  This startup script needs to be run every time the sensors are used. Make the script executable:
  ```
  $ chmod +x startup.sh
  ```
* Enable ssh access using `raspi-config`:
  ```
  $ raspi-config
  ```

## Usage <a name="usage"></a>

### Standalone <a name="standalone"></a>
* Plug in the sensor and check the device file name (Usually it is /dev/video0):
  ```
  $ ls /dev/ | grep video
  ```
* Run the standalone programs as:
  ```
  $ ./simple_blob_tracker4.out CAMERA_NUMBER
  $ ./obj_det_track3.out CAMERA_NUMBER
  ```
  where CAMERA_NUMBER is 0 for /dev/video0

### ROS and Baxter <a name="ros"></a>

#### On Raspberry Pi

* Connect two finger vision sensors to RPi 3 and run `startup.sh` script on RPi:
  ```
  $ ./startup.sh
  ```
  Some times the script may not be able to open both cameras. In this case there will be an `input/output` error warning in the command prompt. Please rerun the script to overcome this error.

#### On Baxter PC

* On the Baxter PC, setup baxter.sh env:
  ```
  $ cd ros_ws/
  $ ./baxter.sh
  ```
* Add lfd_vision packages to ROS_PACKAGE_PATH variable
  ```
  $ cd lfd_vision/
  $ export ROS_PACKAGE_PATH=$ROS_PACKAGE_PATH:`pwd`
  ```
* With this you can start using the nodes:
  ```
  $ roslaunch lfd_vision visual_skin_2fslab11.launch
  ```
  **If there is an error opening the cameras. Check the URLs provided in the config files to match the URL obtained from MJPG-Streamer.** 
