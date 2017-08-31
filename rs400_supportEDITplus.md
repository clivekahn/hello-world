# Software Support Model - Intel® RealSense™ SDK 2.0

## Overview

With the release of Intel® RealSense™ Camera RS400, Intel Perceptual Computing Group is introducing a number of changes to the RealSense™ Cross-Platform API (**librealsense**) support.

Software support for Intel® RealSense™ will be split into two coexisting release lineups: **librealsense 1.x** & **librealsense 2+**.

 * **librealsense 1.x** will continue to provide support for RealSense™ devices: F200, R200 and ZR300.

 * **librealsense 2+** will support the next generation of RealSense™ devices, starting with the RS300 and RS400.


 ## API Changes

**librealsense2** brings the following improvements and new capabilities (which are incompatible with older RealSense devices):

### Streaming API
* **librealsense2** provides a more flexible interface for frames acquisition.  
Instead of a single `wait_for_frames` loop, the API is based on callbacks and queues:
```cpp
// Configure queue of size one and start streaming frames into it
rs2::frame_queue queue(1);
dev.start(queue);
// This call will block the current thread until new frames become available
auto frame = queue.wait_for_frame();
auto pixels = frame.get_data(); // pointer to frame data
```
* The same API can be used in a slightly different way for **low-latency** applications:
```cpp
// Configure direct callback for new frames:
dev.start([](rs2::frame frame){
    auto pixels = frame.get_data(); // pointer to frame data
});
// The application will be notified as soon as new frames become available.
```
**Note:** This approach allows users to bypass buffering and synchronization that was done by `wait_for_frames`.

*  Users who do need to synchronize between different streams can take advantage of the `rs2::syncer` class:
```cpp
auto sync = dev.create_syncer();
dev.start(sync);
// The following call, using the frame timestamp, will block the
// current thread until the next coherent set of frames become available
auto frames = sync.wait_for_frames();
for (auto&& frame : frames)
{
    auto pixels = frame.get_data(); // pointer to frame data
}
```
* `wait_for_frames`is enhanced in this version in that the new API is **thread-safe** by design. You can safely pass a `rs::frame` object to a background thread. This is done without copying the frame data and without extra dynamic allocations.

### Multi-Streaming Model

**librealsense2** eliminates limitations imposed by previous versions with regard to multi-streaming:
* Multiple applications can use librealsense2 simultaneously, as long as no two users try to stream from the same camera endpoint.  
In practice, this means that you can:
  * Stream multiple cameras within a single process
  * Stream camera A from one process and camera B from another process
  * Stream depth from camera A from one process while streaming fisheye / motion from the same camera from another process
  * Stream camera A from one process while issuing controls to camera A from another process
* The following streams of the RS400 act as independent endpoints: Depth, Fisheye, Motion-tracking, Color
* Each endpoint can be exclusively locked using `open/close` methods:
```cpp
// Configure depth to run VGA resolution at 30 frames per second
dev.open({ RS2_STREAM_DEPTH, 640, 480, 30, RS2_FORMAT_Z16 });
// From this point on, device streaming is exclusively locked.
dev.close(); // Release device ownership
```
* Alternatively, users can use the  `rs2::util::config` helper class to configure multiple endpoints at once:
```cpp
rs2::util::config config;
// Declare your preferences
config.enable_all(rs2::preset::best_quality);
// The config object resolves it into concrete camera capabilities
auto stream = config.open(dev);
stream.start([](rs2::frame) {});
```

## Motion-Tracking
Motion-tracking is a first-class citizen in **librealsense2** that supports all the operations generally available to other entities .   
Motion data is acquired using the same APIs as depth and visual data:
```cpp
// Configure the accelerometer to run at 500 frames per second
dev.open({ RS2_STREAM_ACCEL, 0, 0, RS2_FORMAT_MOTION_XYZ32F, 500 });
rs2::frame_queue queue(1);
dev.start(queue);
// Wait for next motion sample
auto frame = queue.wait_for_frame();
auto data = frame.get_data();
auto axes = *(reinterpret_cast<const float3*>(data));
std::cout << axes.x << "," << axes.y << "," <<  axes.z << "\n";
```
**Note:** `rs2::util::config` and `rs2::syncer` can work with motion streams as well.

## New Functionality

* **librealsense2** will be shipped with built-in Python bindings for easier integration.
* New troubleshooting tools are now part of the package, including a tool for hardware-log collection.
* **librealsense2** is capable of handling device disconnects and the discovery of new devices at runtime.

## Transition to CMake
**librealsense2** does not provide hand-written Visual Studio, QT-Creator and XCode project files as you can build **librealsense** with the IDE of your choice using portable CMake scripts.  

## Intel® RealSense™ RS400 and the Linux Kernel

* The Intel® RealSense™ RS400 series (starting with kernel 4.4.0.59) does not require any kernel patches for streaming 
* Advanced camera features may still require kernel patches. Currently, getting **hardware timestamps** is dependent on a patch that has not yet been up-streamed. Without the patch applied you can still use the camera but you will receive a system-time instead of an optical timestamp.
