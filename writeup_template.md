# **Finding Lane Lines on the Road** 
[![Udacity - Self-Driving Car NanoDegree](https://s3.amazonaws.com/udacity-sdc/github/shield-carnd.svg)](http://www.udacity.com/drive)

<img src="examples/laneLines_thirdPass.jpg" width="480" alt="Combined Image" />

Overview
---

When we drive, we use our eyes to decide where to go.  The lines on the road that show us where the lanes are act as our constant reference for where to steer the vehicle.  Naturally, one of the first things we would like to do in developing a self-driving car is to automatically detect lane lines using an algorithm.

In this project you will detect lane lines in images using Python and OpenCV.  OpenCV means "Open-Source Computer Vision", which is a package that has many useful tools for analyzing images.  

To complete the project, two files will be submitted: a file containing project code and a file containing a brief write up explaining your solution. We have included template files to be used both for the [code](https://github.com/udacity/CarND-LaneLines-P1/blob/master/P1.ipynb) and the [writeup](https://github.com/udacity/CarND-LaneLines-P1/blob/master/writeup_template.md).The code file is called P1.ipynb and the writeup template is writeup_template.md 

To meet specifications in the project, take a look at the requirements in the [project rubric](https://review.udacity.com/#!/rubrics/322/view)

# **Project goals** 

---

The goals / steps of this project are the following:
* Make a pipeline that finds lane lines on the road
* Reflect on your work in a written report


[//]: # (Image References)

[image1]: ./examples/gray.jpg "Converted to grayscale"
[image2]: ./examples/blurred.jpg "Blurred"
[image3]: ./examples/edges.jpg "Edges detected"
[image4]: ./examples/region.jpg "Region masked"
[image5]: ./examples/raw_lines.jpg "Example of output result with raw detected lines"
[image6]: ./examples/annotated.jpg "Example of output result with stabilized lines"

---

### Reflection

### 1. Describe your pipeline. As part of the description, explain how you modified the draw_lines() function.

My pipeline consisted of 8 steps.

* First, I convert the image to grayscale
![alt text][image1]

* Then I apply Gaussian blur with kernel size of 5
![alt text][image2]

* I detect edges in the image using Canny Edge Detection algrithm (developed by John F. Canny)
![alt text][image3]
I use values 50 for the low threshold and 150 for the high threshold

* Then I mask region of interest based on the specific image or video clip
![alt text][image4]
I try to be resolution agnostic and calculate the region based on the image size, though adaptations may be required based on the image aspect ration and camera parameters (mounting position, orientation, FoV etc)

* Now I'm ready to run Probabilistic Hough Transform to detect the lines 'cv2.HoughLinesP'.
I use the following parameters:

    rho = 1
    theta = np.pi/180
    threshold = 35
    min_line_length = 30
    max_line_gap = 15

* At this place I've modified default pipline offered in the example and introduced a function to filter and stabilize detected lines over time

    def filter_lines(lines, include_filtered_lines = True, include_raw_lines = False, min_slope = 0.5, max_slope = 2, stabilization_slope_diff = 0.03):
        """
        Input: lines detected by the Hough algorithm
        Output: filtered and stabilized lines on the left and right sides
    
        Pipeline for the lines filtering:
        1. Divide detected lines into left and right according to the slope
        2. Filter detected lines:
            - remove lines with too small or too big slope
            - remove left lines detected on the right side of the image and vice versa
        3. Approximate all the lines with linear regression to find an average line
        4. Filter results of linear regression using mean slope value of history buffer
            - if current slope value differs form the mean slope value for more than specified threshold - skip it
            - otherwise - add current slope into the history buffer
        5. Use mean values (slope, intercept) of the updated history buffer to generate current approximation line
        
        """
After dividing the lines into two groups (left/right) based on the slope, it filters by specified thresholds to reject the lines that are too close to horizontal or vertical (which are not likely to be lane marking lines).
Then the function uses linear least-squares regression to find parameters of the line approximating given set of points from detected lane lines.
History buffer is maintained and can contained specified number of samples for the approximated lines parameters.
This helps to implement additional filtration of the approximated line to exclude the results differ too much from the detection history. By tuning the parameter 'stabilization_slope_diff' you can exclude case of obvious misdetections while maintaining good continuity of the resulting line detection.
Size of the buffer for temporal stabilization can be configured using a helper function 'init_detection'.
By increasing the buffer size, smoothness of the resulting detection is increasing, but that also adds latency to the output visualization.
For the output you can choose to include stabilized and/or raw lines, configure minimum and maximum slope as well as slope difference for filtration (see pictures below).

* New image of a corresponding size is initialized with zeroes and 'draw_lines' function is used to render resulting lines on the left and on the right side.
'draw_lines' function is very simple since all the filtering and stabilization logic is implemented in a separate function ('filter_lines')

* Original image is blended with the new image containing annotated line detections
![alt text][image5]
![alt text][image6]


### 2. Identify potential shortcomings with your current pipeline


Another shortcoming I see is the simple approximation of the lanes using linear regression and producing result consisting of a single line while the lanes can have certain curvature based on the road situation, even in quite limited region of interest.

One potential shortcoming of the pipeline described above is setting the region of interest more or less manually based on a specific image/video.
This makes it difficult to use such an approach for different road situations and camera settings.


### 3. Suggest possible improvements to your pipeline

Possible improvement for the approximation problem would be to sort the line detections by the distance (y-coord), divide then into several segments (2-3, based on number of points and their distribution) and run linear regression on these segments.
This may let to achieve more precise approximation of the lanes curvature.
Even better approach would be to use curve estimation instead of linear regression and then draw the results as curves and not as straight lines.

Another potential improvement could be to include information about the camera parameters into calculation of the region of interest.
In addition, it should be possible to detect horizon line and use this as another input into the region of interest calculation.
On top of that it could be good to define a mask with the hood of the car, if it's visible in the camera picture, to remove it and avoid additional noise in the detections (e.g. due to sun reflection from the hood)
