## Writeup Template

### You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

---

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

[//]: # (Image References)

[image1]: ./output_chessboard_cal/calibrated1.jpg "Undistorted"
[image2]: ./output_step1/straight_lines1_step1_cal.jpg "step1 results"
[image3]: ./output_step2/straight_lines1_step2_binary.jpg "step2 results"
[image4]: ./output_step3/straight_lines1_step3_warp_hist.jpg "step3 results"
[image5]: ./output_step4/straight_lines1_step4_linefit.jpg "step4 results"
[video1]: ./project_video_our.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

SDC_P2 Project report start!!!

# 1. calibration using chessboard images 

The code for this step is contained in the first code cell of the IPython notebook located in "'SDC_P2.ipynb'" 

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]



# 2. Pipeline using test images

## Step1 : Calibrate test image

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]

If want to check the other image results, [click this link](./output_step1)

## Step2 : Binary test image

2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image
  - gradx_binary : kernel size (=3), threshold (20, 100)
  
  
  - grady_binary : kernel size (=3), threshold (60, 100)
  
  
  - meg_binary : kernel size (=3), threshold (30, 180)
  
  
  - dir_binary : kernel size (=3), threshold (0.8, 1.2)
  
  
  - gray_binary : kernel size (=3), threshold (150, 220)  
      gray_binary is used to remove tar marks and tire marks on the lane
      
      
  - hls_binary : threshold (110, 255)
 
 Based on the above results, valid pixels are extracted as follows.

  - condition1 : `(gradx_binary == 1) | (grady_binary== 1) | (mag_binary == 1)`  
    to catch the edge lines any object in images
    
    
  - condition1 : `(gray_binary == 1) & (dir_binary == 1)`  
    to catch the lane line and remove tar marks and tire marks on the lane in image
    
    
  - combined_binary(final) :`(binary_condi1 & binary_condi2) | (hls_binary == 1)`  
    to merge condition1 and condition2 using AND condition, also hls binary is added

Here's an example of my output for this step. 

![alt text][image3]

If want to check the other image results, [click this link](./output_step2)



## Step3 : Warp and Hist test image 


The code for my perspective transform includes a function called `warper()`, The `warper()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  I chose the hardcode the source and destination points in the following manner:

```python
src = np.float32(
    [[(img_size[0] / 2) - 55, img_size[1] / 2 + 100],
    [((img_size[0] / 6) - 10), img_size[1]],
    [(img_size[0] * 5 / 6) + 60, img_size[1]],
    [(img_size[0] / 2 + 55), img_size[1] / 2 + 100]])
dst = np.float32(
    [[(img_size[0] / 4), 0],
    [(img_size[0] / 4), img_size[1]],
    [(img_size[0] * 3 / 4), img_size[1]],
    [(img_size[0] * 3 / 4), 0]])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 585, 460      | 320, 0        | 
| 203, 720      | 320, 720      |
| 1127, 720     | 960, 720      |
| 695, 460      | 960, 0        |



I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

If want to check the other image results, [click this link](./output_step3)

## Step4 : Line fitting test image

Then I did fit my lane lines with a 2nd order polynomial as follows.

  - Using hist plot, find lane's origin (offset to the each lane) within each boundary  
    - If max number of pixel within bound is under threshold (=5), previous step's offset is reused (test image uses only current image)
    
```python
warp_hist_left_bound = warp_histogram[img_size[0]//2 - 400 : img_size[0]//2 - 100]
    warp_hist_right_bound = warp_histogram[img_size[0]//2 + 100 : img_size[0]//2 + 400]
    
    if np.max(warp_hist_left_bound) < 5:
        left_init_offset = left_init_offset_temp
    else:
        left_init_offset = np.argmax(warp_hist_left_bound)
        left_init_offset_temp = left_init_offset
        
    if np.max(warp_hist_right_bound) < 5:
        right_init_offset = right_init_offset_temp
    else:
        right_init_offset = np.argmax(warp_hist_right_bound)
        right_init_offset_temp = right_init_offset
```

  - Among the pixels in the current image [t], extract the pixels in margin based on the lane line of the previous image [t-1] (test image uses only current image)  
  
  
  - Delete outlier pixels using `np.percentile(left_line_diff, 80, interpolation='linear')`  
    - `left_line_diff` is absolute 'x' distance between pixel and lane line [t-1] (test image uses only current image)  
    
  - stacking with extracted pixels in previous step [t-1] (test image uses only current image)
  
  - Calculate temp parameters using `np.polyfit()`   
  
  - Using moving average, calculate final parameter of current image [t] (test image uses only current image)  
    - moving average horizon (=5)
    
  - Convert parameter in image space to parameter in real space
    
  
![alt text][image5]

If want to check the other image results, [click this link](./output_step4)  




# 3. Pipeline using test videos

Through step1 ~ step4 procedure avobe, I tried finding lanes on 'project_video.mp4', 'challenge_video.mp4'

Here's a my video result

Here's a [link to my video result project_video_out](./project_video_out.mp4)  

Here's a [link to my video result challenge_video_out](./challenge_video_out.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

First, this project requires a lot of know-how. This is because you have to find lanes in environmental conditions that can occur in many situations (light conditions, stains on the road, color of road pavement, shadows, etc.). This requires a lot of experience about the threshold value of each filter and the conditions of the combination.

Second, there are many cases in which a valid pixel is not detected depending on the state of the lane. In this case, it is necessary to provide a guaranteed distance through determination (view range) of the corresponding lane.

Finally, it was confirmed that the vibration of values such as road characteristics (curvature, heading angle, offset) was large. Therefore, it is necessary to further apply filtering techniques that can suppress the change of the corresponding values. 
