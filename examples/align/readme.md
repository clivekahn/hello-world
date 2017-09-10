# rs-align Sample

## Overview

This sample demonstrates usage of the `rs2::align` object that allows users to align 2 projected streams.

In this example, we align depth frames to their corresponding color frames and then use the two frames to determine the depth of each color pixel.

Using this information we 'remove' the background (beyond a user defined distance) of the color frame that is further away from the camera.

The example displays a GUI used to control the max distance to show from the original color image.

## Expected Output

The application opens a window and displays a video stream from the camera. 

The window should have the following elements:
- A vertical slider is on the left side of the window to control the depth clipping distance
- A color image with a grayed out background
- A corresponding (colorized) depth image
