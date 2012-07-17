---
layout: main
---
# ardrone_autonomy : A ROS Driver for ARDrone 1.0 & 2.0 {#ardrone_autonomy-:-a-ros-driver-for-ardrone-1.0-&-2.0}

## Introduction {#introduction}

"ardrone_autonomy" is a [ROS](http://ros.org/ "Robot Operating System") driver for [Parrot AR-Drone](http://http://ardrone.parrot.com/parrot-ar-drone/select-site) quadrocopter. This driver is based on official [AR-Drone SDK](https://projects.ardrone.org/) version 2.0 and supports both AR-Drone 1.0 and 2.0. "ardrone_autonomy" is a fork of [AR-Drone Brown](http://code.google.com/p/brown-ros-pkg/wiki/ardrone_brown) driver. This package has been developed in [Autonomy Lab](http://autonomy.cs.sfu.ca) of [Simon Fraser University](http://www.sfu.ca) by [Mani Monajjemi](http://sfu.ca/~mmmonajje). 

## Installation {#installation}

### Pre-requirements {#pre-requirements}

This driver has been tested on Linux machines running Ubuntu 11.10 & 12.04 (32 bit and 64 bit). However it should also work on any other mainstream Linux distribution. Although this package has been tested using ROS "Electric" only, it should work fine with recent ROS "Feurte" as well. The ARDrone SDK has its own build system which usually handles system wide dependencies itself. The ROS package depends on these standard ROS packages: `roscpp`, `image_transport`, `sensor_msgs` and `std_srvs`. 

### Installation Steps {#installation-steps}

The installation follows the same steps needed usually to compile a ROS driver.

* Get the code: Clone (or download and unpack) the driver to your personal ROS stacks folder (e.g. ~/ros/stacks) and `cd` to it. Please make sure that this folder is in your `ROS_PACKAGE_PATH` environmental variable.

	{% highlight bash %}
	$ cd ~/ros/stacks
	$ git clone https://github.com/AutonomyLab/ardrone_autonomy.git
	$ rosstack profile && rospack profile
	$ roscd ardrone_autonomy
	{% endhighlight %}

* Compile the AR-Drone SDK: The driver contains a slightly patched version of AR-Drone 2.0 SDK which is located in `ARDroneLib` directory. To compile it, execute the `./build_sdk.sh`. Any system-wide dependency will be managed by the SDK's build script. You may be asked to install some packages during the installation procedure (e.g `daemontools`). You can verify the success of the SDK's build by checking the `lib` folder.

	{% highlight bash %}
	$ ./build_sdk 
	$ [After a couple of minutes]
	$ ls ./lib
	
	libavcodec.a   libavformat.a    libpc_ardrone_notool.a  libvlib.a
	libavdevice.a  libavutil.a      libsdk.a
	libavfilter.a  libpc_ardrone.a  libswscale.a
	{% endhighlight %}

* Compile the driver: You can easily compile the driver by using `rosmake ardrone_autonomy` command.

## How to Run {#how-to-run}

The driver's executable node is `ardrone_driver`. You can either use `rosrun ardrone_autonomy ardrone_driver` or put its name in a custom launch file.

## Reading from AR-Drone {#reading-from-ar-drone}

### Navigation Data {#navigation-data}

Information received from the drone will be published to the `ardrone/navdata` topic. The message type is `ardrone_autonomy::Navdata` and contains the following information:

* `header`: ROS message header
* `batteryPercent`: The remaining charge of the drone's battery (%)
* `state`: The Drone's current state: 
	* 0: Unknown
	* 1: Inited
	* 2: Landed
	* 3,7: Flying
	* 4: Hovering
	* 5: Test (?)
	* 6: Taking off
	* 8: Landing
	* 9: Looping (?)
* `rotx`: Left/right tilt in degrees (rotation about the X axis)
* `roty`: Forward/backward tilt in degrees (rotation about the Y axis)
* `rotz`: Orientation in degrees (rotation about the Z axis)
* `altd`: Estimated altitude (cm)
* `vx`, `vy`, `vz`: Linear velocity (mm/s) [TBA: Convention]
* `ax`, `ay`, `az`: Linear acceleration (g) [TBA: Convention]
* `tm`: Timestamp of the data returned by the Drone

### Cameras {#cameras}

Both AR-Drone 1.0 and 2.0 are equipped with two cameras. One frontal camera pointing forward and one vertical camera pointing downward. This driver will create three topics for each drone: `ardrone/image_raw`, `ardrone/front/image_raw` and `ardrone/bottom/image_raw`. Each of these three are standard [ROS camera interface](http://ros.org/wiki/camera_drivers) and publish messages of type [image transport](http://www.ros.org/wiki/image_transport). 

* The `ardrone/*` will always contain the selected camera's video stream and information.

* The way that the other two streams work depend on the type of Drone.

	* Drone 1

	Drone 1 supports four modes of video streams: Front camera only, bottom camera only, front camera with bottom camera inside (picture in picture) and bottom camera with front camera inside (picture in picture). According to active configuration mode, the driver decomposes the PIP stream and publishes pure front/bottom streams to corresponding topics. The `camera_info` topic will include the correct image size. 

	* Drone 2

	Drone 2 does not support PIP feature anymore, therefore only one of `ardrone/front` or `ardrone/bottom` topics will be updated based on which camera is selected at the time.

### Tag Detection {#tag-detection}

The `Navdata` message also returns the special tags that are detected by the Drone's on-board vision processing system. To learn more about the system and the way it works please consult AR-Drone SDK 2.0's [developers guide](https://projects.ardrone.org/projects/show/ardrone-api/). These tags are being detected on both drone's video cameras on-board at 30fps. To configure (or disable) this feature look at the "Parameters" section in this documentation.

The detected tags' type and position in Drone's camera frame will be published to the following variables in `Navdata` message:

* `tags_count`: The number of detected tags.
* `tags_type[]`: Vector of types of detected tags (details below)
* `tags_xc[]`, `tags_yc[]`, `tags_width[]`, `tags_height[]`: Vector of position components and size components for each tag. These numbers are expressed in numbers between [0,1000]. You need to convert them back to pixel unit using the corresponding camera's resolution (can be obtained fron `camera_info` topic). 
* `tags_orientation[]`: For the tags that support orientation, this is the vector that contains the tag orientation expressed in degrees [0..360).

By default, the driver will configure the drone to look for _oriented roundels_ using bottom camera and _2D tags v2_ on indoor shells (_orange-yellow_) using front camera. For information on how to extract information from `tags_type` field. Check the FAQ section in the end.

## Sending Commands to AR-Drone {#sending-commands-to-ar-drone}

The drone will *takeoff*, *land* or *emergency stop/reset* by publishing an `Empty` ROS messages to the following topics: `ardrone/takeoff`, `ardrone/land` and `ardrone/reset` respectively.

In order to fly the drone after takeoff, you can publish a message of type [`geometry_msgs::Twist`](http://www.ros.org/doc/api/geometry_msgs/html/msg/Twist.html) to the `ardrone/cmd_vel` topic. 

	-linear.x: move backward
	+linear.x: move forward
	-linear.y: move right
	+linear.y: move left
	-linear.z: move down
	+linear.z: move up
	
	-angular.z: turn left
	+angular.z: turn right

The range for each component should be between -1.0 and 1.0. The maximum range can be configured using ROS parameters discussed later in this document. Publishing "0" values for all components will make the drone keep hovering.

## Services {#services}

Calling `ardrone/togglecam` service with no parameters will change the active video camera stream.

## Parameters {#parameters}

The parameters listed below are named according to AR-Drone's SDK 2.0 configuration. Unless you set the parameters using `rosparam` or in your `lauch` file, the default values will be used. These values are applied during driver's initialization phase. Please refer to AR-Drone SDK 2.0's [developer's guide](https://projects.ardrone.org/projects/show/ardrone-api/) for information about valid values.

* `bitrate_ctrl_mode` - default: DISABLED
* `max_bitrate` - (AR-Drone 2.0 only) Default: 4000 Kbps
* `bitrate` -  Default: 4000 Kbps
* `outdoor` - Default: 0
* `flight_without_shell` - Default: 1
* `altitude_max` - Default: 3000 mm
* `altitude_min` - Default: 100 mm
* `control_vz_max` - Default: 850.0 mm/s
* `control_yaw` - Default: 100 degrees/?
* `euler_angle_max` - Default: 12 degrees
* `navdata_demo` - Default: 1
* `detect_type` - Default: `CAD_TYPE_MULTIPLE_DETECTION_MODE` (TBA)
* `enemy_colors` - Default: `ARDRONE_DETECTION_COLOR_ORANGE_YELLOW` (TBA)
* `enemy_without_shell` - Default: 1
* `detections_select_h` - Default: `TAG_TYPE_MASK(TAG_TYPE_SHELL_TAG_V2)` (TBA)
* `detections_select_v_hsync` - Default: `TAG_TYPE_MASK(TAG_TYPE_BLACK_ROUNDEL)` (TBA)

## License {#license}

The Parrot's license, copyright and disclaimer for `ARDroneLib` are included with the package and can be found in `ParrotLicense.txt` and `ParrotCopyrightAndDisclaimer.txt` files respectively. The other parts of the code are subject to `BSD` license.

## FAQ {#faq}

### How can I report a bug, submit patches or ask for a request? {#how-can-i-report-a-bug,-submit-patches-or-ask-for-a-request?}

`github` offers a nice and convenient issue tracking and social coding platform, it can be used for bug reports and pull/feature request. This is the preferred method. You can also contact the author directly. 

### Why the `ARDroneLib` has been patched? {#why-the-`ardronelib`-has-been-patched?}

The ARDrone 2.0 SDK has been patched to 1) Enable the lib only build 2) Make its command parsing compatible with ROS and 3) To fix its weird `main()` function issue

### How can I extract camera information and tag type from `tags_type[]`? {#how-can-i-extract-camera-information-and-tag-type-from-`tags_type[]`?}

`tag_type` contains information for both source and type of each detected tag. In order to extract information from them you can use the following c macros and enums (taken from `ardrone_api.h`)

{% endhighlight %}c++
#define DETECTION_EXTRACT_SOURCE(type)  ( ((type)>>16) & 0x0FF ) {#detection_extract_source(type)--(-((type)>>16)-&-0x0ff-)}
#define DETECTION_EXTRACT_TAG(type)     ( (type) & 0x0FF ) {#detection_extract_tag(type)-----(-(type)-&-0x0ff-)}

typedef enum
{
  DETECTION_SOURCE_CAMERA_HORIZONTAL=0,   /*<! Tag was detected on the front camera picture */
  DETECTION_SOURCE_CAMERA_VERTICAL,       /*<! Tag was detected on the vertical camera picture at full speed */
  DETECTION_SOURCE_CAMERA_VERTICAL_HSYNC, /*<! Tag was detected on the vertical camera picture inside the horizontal pipeline */
  DETECTION_SOURCE_CAMERA_NUM,
} DETECTION_SOURCE_CAMERA;

typedef enum
{
  TAG_TYPE_NONE             = 0,
  TAG_TYPE_SHELL_TAG        ,
  TAG_TYPE_ROUNDEL          ,
  TAG_TYPE_ORIENTED_ROUNDEL ,
  TAG_TYPE_STRIPE           ,
  TAG_TYPE_CAP              ,
  TAG_TYPE_SHELL_TAG_V2     ,
  TAG_TYPE_TOWER_SIDE       ,
  TAG_TYPE_BLACK_ROUNDEL    ,
  TAG_TYPE_NUM
} TAG_TYPE;

{% endhighlight %}

## TODO List {#todo-list}

* Enrich `Navdata` with magneto meter and baro meter information
* Add separate topic for drone's debug stream (`navdata_demo`)
* Add the currently selected camera name to `Navdata`
* Make the `togglecam` service accept parameters
