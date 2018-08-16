[//]: # (Image References)

[image1]: ./output_images/step0/Undistort.jpg "Undistorted"
[image2]: ./output_images/step0/Undistort_test5.jpg "Undistorted"
[image3]: ./output_images/step1/binary.jpg "Binary Example"
[image4]: ./output_images/step1/birdseye.jpg "Warp Example"
[image5]: ./output_images/step1/roi.jpg "Region of interest"
[image6]: ./output_images/step1/separate_binary_lines.jpg "Separate Lines"
[image7]: ./output_images/step1/project_test5.jpg "Projected lines"
[video1]: ./processed_project_video.mp4 "Video"

# Advanced Lane Line Finding

It's a lane line detector that can detect lane line in different lights condition.

It's a Udacity self-driving car Nanodegree and it have some requirements to be completed. Please see the links below for more details.

- [![Udacity - Self-Driving Car NanoDegree](https://s3.amazonaws.com/udacity-sdc/github/shield-carnd.svg)](http://www.udacity.com/drive)
- [requirements/rubric](https://review.udacity.com/#!/rubrics/571/view)

# Introduction
This project are divided in N stages:  

- Camera calibration.
- Distortion corrected image.
- Binary lane line image using gradient and color transforms.  
- Perspective Transform to bird's eye view.
- Identifying lane line pixels using sliding windows.
- Extracting the local curvature of the road and vehicle localization.
- Projecting the detected lane lines onto the original image.
- Video Processing Pipeline.

---

### Before you continue:

The images for camera calibration are stored in the folder called `camera_cal`.  Images in `test_images` are for testing the pipeline on single frames.  Results are in the  `ouput_images` folder and subfolders `step0, step1, step2`.

---

# Let's start

In the following I will describe how I did each point in the implementation. 

![Not important Gif](https://media.giphy.com/media/1TzKVQwH820wM/giphy.gif)


# Camera calibration 
![Camera calibration effect gif](https://media.giphy.com/media/JxP0kMdpR3ORa/giphy.gif)

To Detect curvature of lane lines is crucial to work with images that are distortion corrected. But how can I do it?

Get images of chessboard with the same camera that you recorded the lane lines, so we will get the chessboard patterns and distort it, so we will apply this same distortion in the video are recorded in the car to get the lane lines images/videos.
In our case, we will apply it using the images that are in `camera_cal` folder.

![Undistort][image1]

You will find the code for `distortion correction` inside the IPython notebook located in "./step0_camera_calibration.ipynb" .  

Starting by creation of "object points", which are (x, y, z) coordinates of the chessboard corners. In the second Cell have a `points` variable, that's just a replicated array of coordinates, and `objpoints` variable will be appended with a copy of it every time I successfully detect all chessboard corners in a test image. In the same Cell, have a  `imgpoints` variable that will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

`objpoints` and `imgpoints` are then used to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function. I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 


# Test Image Pipeline

## Example of a distortion corrected image
It's the result of applying the undistortion transformation to a test image using the distortion coefficients given a set of chessboard imagehessboard corners distortion 

(left distorted, right corrected)
![Undistort][image2]
## Binary lane line image using gradient and color transforms

For color thresholding I worked in HLS space. Only the L and S channel were used. I used the s channel for a gradient filter along x and saturation threshold, as well as the l channel for a luminosity threshold filter. A combination of these filters
is used in the function `binarize` in the file `step1_test_image_pipeline.ipynb`. The binarized version of the image above looks as follows
![Binarized image][image3]

## Perspective Transform to bird's eye view
![Just a reandom gif, it's not important](https://media.giphy.com/media/8WLBQsjVNh3wc/giphy.gif)

A perspective transform to and from "bird's eye" perspective is done in a function called `warp()`, which appears in 5th cell of the notebook `step1_test_image_pipeline.ipynb`.  The `warp()` function takes as input an color image (`img`), as well as the `tobird` boolean paramter. The parameters `src` and ` dst` of the transform are hardcoded in the function as follows:

```
    corners = np.float32([[190,720],[589,457],[698,457],[1145,720]])
    new_top_left=np.array([corners[0,0],0])
    new_top_right=np.array([corners[3,0],0])
    offset=[150,0]
    
    img_size = (img.shape[1], img.shape[0])
    src = np.float32([corners[0],corners[1],corners[2],corners[3]])
    dst = np.float32([corners[0]+offset,new_top_left+offset,new_top_right-offset ,corners[3]-offset])    
```
This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 190, 720      | 340, 720    | 
| 589, 457      | 340, 0      |
| 698, 457      | 995, 0      |
| 1145,720      | 995, 720    |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image. See the following image
![alt text][image4]
Additionally, I included a region of interest that acts on the warped image to reduce artefacts at the bottom of the image.
This region is defined through the function `region_of_interest()` and was tested using the wrappers `warp_pipeline(img)` and `warp_binarize_pipeline(img)`An example is shown below. 
![alt text][image5]

## Identifying lane line pixels using sliding windows
The function `find_peaks(img,thresh)` in cell of `step1_tes_images_pipeline.ipynb` takes the bottom half of a binarized and warped lane image to compute a histogram of detected pixel values. The result is smoothened using a gaussia filter and peaks are subsequently detected using. The function returns the x values of the peaks larger than `thresh` as well as the smoothened curve. 

Then I wrote a function `get_next_window(img,center_point,width)` which takes an binary (3 channel) image `img` and computes the average x value `center` of all detected pixels in a window centered at `center_point` of width `width`. It returns a masked copy of img a well as `center`.

The function `lane_from_window(binary,center_point,width)` slices a binary image horizontally in 6 zones and applies `get_next_window`  to each of the zones. The `center_point` of each zone is chosen to be the `center` value of the previous zone. Thereby subsequent windows follow the lane line pixels if the road bends. The function returns a masked image of a single lane line seeded at `center_point`. 
Given a binary image `left_binary` of a lane line candidate all properties of the line are determined within an instance of a `Line` class. The class is defined in cell 11
``` 
    left=Line(n)
    detected_l,n_buffered_left = left.update(left_binary)
```
The `Line.update(img)` method takes a binary input image `img` of a lane line candidate, fits a second order polynomial to the provided data and computes other metrics. Sanity checks are performed and successful detections are pushed into a FIFO que of max length `n`. Each time a new line is detected all metrics are updated. If no line is detected the oldest result is dropped until the queue is empty and peaks need to be searched for from scratch. 

A fit to the current lane candidate is saved in the `Line.current_fit_xvals` attribute, together with the corresponding coefficients. The result of a fit for two lines is shown below.

![line fit][image6]

## Extracting the local curvature of the road and vehicle localization

The radius of curvature is computed upon calling the `Line.update()` method of a line. The method that does the computation is called `Line.get_radius_of_curvature()`. The mathematics involved is summarized in [this tutorial here](http://www.intmath.com/applications-differentiation/8-radius-curvature.php).  
For a second order polynomial f(y)=A y^2 +B y + C the radius of curvature is given by R = [(1+(2 Ay +B)^2 )^3/2]/|2A|.

The distance from the center of the lane is computed in the `Line.set_line_base_pos()` method, which essentially measures the distance to each lane and computes the position assuming the lane has a given fixed width of 3.7m. 

## Projecting the detected lane lines onto the original image

THis step is implemented in cells 13 and 14 in `step1_test_image_pipeline.ipynb` in the function `project_lane_lines()` which is called by `process_image()`. This last function process_image(img) handles all lost lane logic. Here is an example of the result on a test image:

![alt text][image7]


---

# Video Processing Pipeline

Finally, I took the code developed in `step1_test_image_pipeline.ipynb` and processed the project video in the notebook  `step2_video_pipeline.ipynb`. The processed project video can be found here:
[link to my video result](./processed_project_video.mp4)

---

# Discussion

## Problems encountered and Outlook

Getting the pipeline to be robust against shadows and at the same time capable of detecting yellow lane lines on white ground was the greates difficulty. I took the approach that lines should never have very low saturation values, i.e. be black. Setting a minimal value for the saturation helped when paired with the x gradient and absolute gradient threshold. Problematic is also the case when more than two lanes get detected. For lines detected far away a threshold on the distance eliminated the problem. By far the most work was implementing the logic of a continuous update of detected lines as well as restarting when the buffer of previous lines emptied. At the moment the pipeline will likely fail as soon as more (spurious) lines are on the same lane, as e.g. in the first challenge video. This could be solved by building separate lane line detectors for yellow and white together with additional logic which line to choose. 
