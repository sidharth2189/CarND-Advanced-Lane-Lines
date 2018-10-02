
**Advanced Lane Finding Project**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

The code file is P2.ipynb

[//]: # (Image References)

[image1]: ./output_images/undistorted.jpg "Undistorted"
[image2]: ./test_images/test5.jpg "Road Image"
[image3]: ./output_images/Undistorted_test5.jpg "Distortion Corrected"
[image4]: ./output_images/threshold_test5.jpg "Binary Example"
[image5]: ./output_images/perspective.jpg "Warp Example"
[image6]: ./output_images/sliding_window_polyfit.jpg "Fit Visual"
[image7]: ./output_images/ProjectedLane.jpg "Output"
[video1]: ./Output_project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the second code cell of the IPython notebook located in "P2.ipynb". 

I started by importing all the calibration images of chessboard using the glob api. I also define the number of inside corner points in a row and a column of the chessboard. In the next step, I prepared "object points", which would be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `obj_points` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image, which would be image points. Image points are detected using the function `cv2.findChessboardCorners()`. These `img_points` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `obj_points` and `img_points` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

The camera matrix and distortion coefficients calculated from camera calibration step, were used to apply distortion correction to one of the test images like this one: 
![alt text][image2]

Using `cv2.undistort(image, mtx, dist, None, mtx)`, the test image was distortion corrected. After distortion correction, the above test image looks as below
![alt text][image3]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image in cell 8 of the code file.  
The distortion corrected image was first converted into HLS color space. The sobe operator `cv2.Sobel()` was applied to this image to calculate gradient in x-direction. Regions inside the binary threshold range `sx_thresh=(20, 100)` were retained and the rest was blacked out. Further, the s-channel in the image was similarly thresholded in a binary range `s_thresh=(170, 255)`

`sxbinary[(scaled_sobel >= sx_thresh[0]) & (scaled_sobel <= sx_thresh[1])] = 1
s_binary[(s_channel >= s_thresh[0]) & (s_channel <= s_thresh[1])] = 1

combined_binary = np.zeros_like(sxbinary)
combined_binary[(s_binary == 1) | (sxbinary == 1)] = 1`

The binary thresholded image was region masked to keep the areas where lanes are found in the image using `cv2.fillPoly()` and `cv2.bitwise_and()` in the mask function in cell 7 of the code file.

```python
left = (100, img.shape[0])
    apex1 = (600, 400)
    apex2 = (700, 400)
    right = (1150, img.shape[0])
    vertices = np.array([[left, apex1, apex2, right]], dtype=np.int32)
    
 combined_binary = mask_area(combined_binary, vertices)
 ```

Here's an example of my output for this step.
![alt text][image4]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `perspectiveTF()`, which appears in cell 10 of the code file.  The  function `perspectiveTF()` takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points. The source and destination points are chosen in the following manner:

```python
src = np.float32([[180,image.shape[0]],[600,450],[700,450],[1130,image.shape[0]]])
dst = np.float32([[320,image.shape[0]],[320,0],[950,0],[950,image.shape[0]]])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 600, 450      | 320, 0        | 
| 180, 720      | 320, 720      |
| 1130, 720     | 950, 720      |
| 700, 450      | 950, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a binary thresholded version of a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image5]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

After applying calibration, thresholdin and masking, and a perspective transform to a road image, I have a binary image where the lane lines stand out clearly. In the next step, I decide explicitly decide which pixels are part of the lines and which belong to the left line and which belong to the right line.

I start by creating a histogram of where the peaks of summed y-values occur across the image in x-direction for the binary thresholded image. The two highest peaks from the histogram serve as a starting point for determining where the lane lines are, and then I use sliding windows moving upward in the image (further along the road) to determine where the lane lines go. this is coded in cell 11 0f the code file in the function `find_lane_pixels()`. This function takes in the perspective transformed image and finds lane pixels for left and right lanes. The next step is to set a few hyperparameters related to the sliding windows, and set them up to iterate across the binary activations in the image.

The method has following steps to iterate in order to track curvature :

1) Loop through each window in nwindows
2) Find the boundaries of our current window. This is based on a combination of the current window's starting point (leftx_current and rightx_current), as well as the margin you set in the hyperparameters.
3) Use cv2.rectangle to draw these window boundaries onto our visualization image out_img. This is required for the quiz, but you can skip this step in practice if you don't need to visualize where the windows are.
4) Now that we know the boundaries of our window, find out which activated pixels from non zero y pixel valued and non zero x pixel valued co-ordinates actually fall into the window.
5) Append these to our lists left lane indices and right lane indices.
6) If the number of pixels you found in Step 4 are greater than a minimum pixel value set inside hyperparameters, re-center our window based on the mean position of these pixels.

Then I fit my lane lines with a 2nd order polynomial using the `np.polyfit function` and plot the resulting polynomial on the image.

![alt text][image6]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

In the function `measure_curvature_pixels()` in cell 14 of the code file, I calculate the vehicle position, left and right curvature values. 

The curvature calculation can be done as show [here](https://classroom.udacity.com/nanodegrees/nd013/parts/edf28735-efc1-4b99-8fbb-ba9c432239c8/modules/5d1efbaa-27d0-4ad5-a67a-48729ccebd9c/lessons/626f183c-593e-41d7-a828-eda3c6122573/concepts/2f928913-21f6-4611-9055-01744acc344f)

The curvature is calculated at the maximum Y-value. Convert the value from pixels to metres for the polynomial fit obtained previously.
For the vehicle position, the left and right position of x at max Y-value are calculated so that the lane mid-point can be fetched. The deviation of the midpoint of the lane from the center of the image is the offset we're looking for.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in cell 15 of the code file, in the function `draw_Lane`.  Here is an example of my result on a test image:

![alt text][image7]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./Output_project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  
