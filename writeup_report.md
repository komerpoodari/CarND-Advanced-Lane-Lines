## Writeup Report: Advanced Lane Finding Project / Udacity CarND Program

### This report describes the implementation of Advanced Lane Finding Project pipe line and other required processing and computations.

---

**Advanced Lane Finding Project**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients, given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the "birds-eye view" detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1]: ./examples/undist_calib3.png "Undistorted"
[image2]: ./examples/undist_example.png "Road Transformed"
[image3]: ./examples/undist_combo_test1.png "test1 image & its undistorted, thresholded binary"
[image4]: ./examples/undist_combo_xfm_test3.png "Warp Example"
[image5]: ./examples/test6-linefit.png "Fit Visual"
[image6]: ./examples/painted_lane_test5.png "Output"


## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe in this report how I addressed each point in my implementation.  


### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first code cell of the IPython notebook located in **"./Adv-lane-find.ipynb" (section 1. Camera Calibration)**.  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  Then I stored the output of calibration function in a pickle file, so the information can be used later in pipe line processing, by extracting  camera matrix, and distortion coefficients from the pickle file. 

I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 
![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

I applied distortion correction on a road image by invoking `cv2.undistort()`, with camera matrix, distortion co-efficients obtained from camera calibration (and stored in a pickle file) and original image as inputs and obtained undistorted image as output, as pasted below.
![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image in **"./Adv-lane-find.ipynb" (section 2. Thresholding)**. Though I did not use all the implemented threshold functions, I wanted to get a sense of these different thresholding techniques.  Here's an example of my output for this step. Finally I ended up using  HLS space color thresholds masked with relevant portion of the image section.  The relevant logic is implemented in functions `region_of_interest()` and `combo_threshold()` in **"./Adv-lane-find.ipynb" (section 2. Thresholding)**. I had to experiment with the thresholds for channels 'S' and 'H' with trial and error to detect yellow and white lines. I took guidance from forums and Internet with starting and fine tuning values. After getting the whole binary image after thresholding, I applied a mask with relevant trepizoid box, encompassing road lanes. This logic in function `region_of_interest()` in the notebook section.

Here is the thresholded binary image output, along with original image.
![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `do_pxform ()`, which appears in **"./Adv-lane-find.ipynb" (section 3. Perspective Transform)**.  The `do_pxform ()` function takes as inputs an image (`image`). I chose the hardcode the source and destination points in the following manner:

```python
#....
 x_offset = 250
#....
    #guidance from 'straight_lines1.jpg' image
    src = np.float32([[235, 690],[579, 460],[705, 460],[1060, 690]])  

    #position it in the middle of the frame ractagle, i.e.  botton boundaries are
    dst = np.float32([[x_offset, 720],[x_offset, 0],[nx - x_offset, 0],[nx- x_offset, 720]])
```

This resulted in the following source and destination points:

| Source        | Destination   | Point Location |
|:-------------:|:-------------:|:---------------| 
| 235, 690      | 250, 720      | bottom-left    |
| 579, 460      | 250, 0        | top-left       |
| 705, 460      | 1030, 0       | top-right      |
| 1060, 690     | 1030, 720     | bottom-right   |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.
Here is the the snapshot raw test and its counter-part undistorted, thresholded and perspective transformed (warped) 'birds-eye view' binary image. I had to experiment and fine-tune the top-right and top-left corners in 'src' points.
![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Then I implemented line fitting on identified lanes in undistorted, thresholded and warped ('birds-eye view') binary images in **"./Adv-lane-find.ipynb" (section 4. Polynomial Line Fitting)**. The functions `sliding_windows_polyfit()` and `incremental_polyfit()` fit the second order polynomials from scratch and incrementally respectively.

I selected 12 windows and adjusted down the margin and minimum pixel values accordingly in `sliding_windows_polyfit()`. The lines were reasonable fit.

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I implemented function `compute_radc_centerd()` in **"./Adv-lane-find.ipynb" (section 5. Compute Lanes Radius of Curvature and Vehicle Position)**.  The function uses the binary image, lines fit for left and right lanes and the indices for non-zero pixels of left and right lanes as inputs and computes the radius of curvature for left and right lanes and also vehicle position from center. As per the guidance given in the class lectures I used the formulas to fit lines in real world then used the line fittings to compute the radii of curvature for left and right lanes.

I implemented logic to compute the vehicle position by using the  X-intercepts of the left & right lane line fittings, basically the difference between image x-positio center (vehicle position) and the x-intercepts (lane center).

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in the function `paint_lane()`, in the **"./Adv-lane-find.ipynb" (section 6. Painting The Lane in the Front)**. The function superimposes with green color the identified lane area in the front of the vehicle and displays the radius of curvature of lane and vehicle position relative to lane center. The function uses original image, binary warped image, inverse matrix of perspective transform, lane line fittings and the radius of curvature and vehicle position data in real word units.  Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Then in **"./Adv-lane-find.ipynb" (section 7.  Video Processing)** section I implemented higher level function `process_image()` that gets invoked video `fl_image()` function. In this section implemented the class 'Line', based on the course material guidance and 'process_image()' function. I averging the Line fit polynomial coefficients over 5 most recent frames. I invoked incremental polynomial fitting, if most recent fit was fine within some sanity checks such as X-intercept distances. and co-efficient changes. Overall the processing pipeline worked reasonably well.

The output video file is *"./project_output_video.mp4"*. Here's a link to my output video on YouTube.
**https://youtu.be/5ZMIssN6EX4**
---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

I faced the following problems during the course of project implementation.

**1. Thresholding**
Selecting right combination of thresholding techniques is very critcial for identifying the lane pixels. I experimented with multiple combinations and ended up with a combination of 'S' and 'L' combination thresholding in HLS color space.  I would furher experiment when I get opportunity with other thresholding combinations. Especially I encountered problems during processing the road segments that are sunny and involve bridges. The pavement color on the bridges is bright and sunny.

**2. Sanity Checks**
In processing the image in the function `process_image()` I had to play with sanity check parameters such as x-intercepts of the lanes and line fitting co-efficients.  But these do not matter much as long as the line fittings are correct.

I felt it is a lengthy project, given my personal situation of multitasking.  In the end, it's good to see a lane of the familiar 280 Highway painted in green.
