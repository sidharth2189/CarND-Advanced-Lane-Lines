# CarND-Advanced-Lane-Lines-P2

A pipeline for lane identification that identifies the lane boundaries in a video from a front-facing camera on a car. Some information regarding this repository as below.

1) 'camera_cal' folder contains chessboard images for calibration.

2) 'test_images' folder contains single frames (test images) from the project video for testing on the pipeline

3) 'output_images' folder contains a sample undistorted chessboard image. The folder also contains corresponding images for test images after each of the following operations:
  a) Undistorted
  b) Masked after being color and gradient thresholded
  c) Perspective transformed
  d) Sliding window and polynomial fit implementation

4) P2.ipynb is the project code file

5) Output_project_video.mp4 is the output file from testing the pipeline on project_video.mp4

6) write_up.md explains how the project objectives were approached.

Reference Repository: https://github.com/udacity/CarND-Advanced-Lane-Lines
