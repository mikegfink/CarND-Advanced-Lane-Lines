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

[image1]: ./output_images/calibration2.jpg "Chessboard"
[image2]: ./output_images/undist_test2.jpg "Undistorted Road"
[image3]: ./test_images/test2.jpg "Original Road"
[image4]: ./output_images/binary_test2.jpg "Binary Example"
[image5]: ./output_images/warped_test2.jpg "Warp Example"
[image6]: ./output_images/test2.jpg "Detected Road"
[image7]: ./output_images/color_fit_lines.jpg "Fit Visual"
[image8]: ./examples/example_output.jpg "Output"
[video1]: ./project_video.mp4 "Video"

---

### Writeup / README

### Camera Calibration

The code for this step is contained in the third code cell of the IPython notebook located in "./adv_lane_finding.ipynb".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

An example detection can be seen in this image:
![Chessboard Detection][image1]

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test images using the `cv2.undistort()` function and obtained this result: 

![Undistorted image][image2]
![Original image][image3]

### Pipeline (single images)

#### 1. Lane Detection Thresholding

I used a combination of color and gradient thresholds to generate a binary image (thresholding steps the `pipeline` function of ./adv_lange_finding.ipynb). Specifically, I used a combination of three different binary images. 

First, I applied a yellow and white filter to select for yellow-ish and white-ish pixels only. This was grayscaled and then a directional gradient applied for lines that were within reasonable bounds on an unwarped image (not quite vertical, not horizontal).

Second, I used the saturation channel of the image, after converting from RGB to HLS, and applied combined directional and magnitude threshold. If both the directional and magnitude conditions were met, then the pixel was accepted for the final image. This was useful to supplement the yellow and white filter, and picked up much of the same pixels. It provided coverage for darker image portions because no absolute threshold was applied, so differences between the road and shadowed lines were picked up more easily.

Third, I used the lightness channel with a raw value threshold and mostly a Sobel filter in the x direction. This helped pick up lines missed by the yellow and white filter.

Here's an example of my output for this step (it corresponds to the same image in the Camera Calibration section above).

![Binary Image][image4]

#### 2. Performing a perspective transform.

The code for my perspective transform includes a function called `warper` in the IPython notebook.  The `warper` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  I chose to hardcode the source and destination points in the following manner:

```python
    shape = img.shape
    lx1 = shape[1] * 0.443
    lx2 = shape[1] * 0.182
    rx1 = shape[1] * 0.558
    rx2 = shape[1] * 0.858
    y1 = shape[0] * 0.65
    y2 = shape[0] * 0.99
    return np.float32([[lx1,y1],[rx1,y1],[lx2,y2],[rx2,y2]])

    x1 = shape[1] * 0.3
    x2 = shape[1] * 0.7
    y1 = -200
    y2 = shape[0]
    dst = np.float32([[x1,y1],[x2,y1],[x1,y2],[x2,y2]])
```

These were matched from some of the example images and, for the most part arbitrarily chosen for the destination points. The most interesting decision was to use -200 as the upper vertical destination. This is because the points further away in an image were emphasized too much by the transform, this reduced that effect by stretching the image. Another way to reduce this would have been to change the source area, but since that requires recalculation, this is simpler.

Below is an example of the warped image we've been using throughout.

![Warped Binary][image5]

#### 3. Identifying lane-line pixels and fitting their positions with a polynomial.

Detecting the lane lines used a combination of windowed pixel detection and, for the video, polynomial margin detection.

Windowed pixel detection involves splitting the image into horizontal slices, say 20, with a given width, say 120 pixels. Then, we check for pixels that are present in each of the windows. Specifically, we determine the first window location based on the two x values that contain the most pixels via a histogram across the width of the image. If we have detected any lane lines in previous frames, then those are used instead of the histogram method. Each subsequent window is centered on the mean pixel location of the previous window. That is, unless a linear fit of the pixels in the window indicates that the line will exceed the window bounds to the left or right. In that case, the window may be placed on the right (or left) instead of above the previous window. This is to handle curved lanes. The alternate method only came into play on the harder challenge video, and the pipeline was not successful there for other reasons. This method can be found in the `windowed_pixels` function. The polynomial search is simpler and can be found in `search_around_poly`.

Once pixels for each lane line have been detected, we just make a `np.polyfit` call for an order 2 polynomial. Once the fit is complete we do some validation to check that the lines are not too different from one another and not too different from the previously accepted line from previous frames. Several heuristics were used for this including raw difference in x values at the bottom of the image and detecting too great a change in the second order coefficient, which guides the direction of the curve and how wide it is. In addition, running averages of the lane lines for the video were kept in deques, which allowed for easy removal of the oldest value. Previous detections were kept for a configurable number of frames before reverting to the initial detection method.

In the beginning, I actually used a single line by translating the points to the center of the image as a simplification. This worked perfectly on the project video and would allow the system to compensate for faulty lines on the left or the right. However, I realized that this would cause problems with turns of any significance, since it would be a simple x translation to line up the lane lines. I suspec that this approach could still work by applying a transformation to uncurve each lane line, then shifting them to the center. However, I decided to revert to tracking each line separately for simplicity.

![Warped Fit][image5]

#### 4. Calculating the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in `measure_curvature`. The polynomial coefficients were converted to meters via a pixels to meters approximation, then plugged into the polynomial radius function.

The center of the vehicle was assumed to be the center of the camera. As a result the distance from the center of the lane was calculated by the difference between the center of the image and the center of the lane (given by the difference between the lane positions at the bottom of the image).

#### 5. Final combined image.

Combining all the previous steps, plotting the detected area, unwarping the lane detection images and printing the curvature and distance to center results in an image like the following:

![Fit Image][image6]

---

### Pipeline (video)

#### 1. Video output

Here's a [link to my video result](./test_videos_output/project_video.mp4). The challenge video also tracks relatively well [here](./test_videos_output/challenge_video.mp4). I left the harder challenge video output in the same folder, though the pipeline before poorly there.

---

### Discussion

#### 1. Problems, improvements?

The primary problem faced during implementation was detecting the lane lines in the various lighting conditions. Finding some type of balance between detecting too many pixels and too few is very challenging, especially for the challenge videos. I have a feeling that increased image resolution could help here. The approach I took was to try to find color channels and regions of those color channels that would detect lines in some conditions but not others. For example, trying to find a way to detect low light lane lines without lighting up every pixel in a very bright image. One thing I did not try that could have helped would be to normalize all the pixels towards a mean value, lightening the darks and darkening the lights. Whether that would wash out too much information or not, I'm not sure, but perhaps it could provide an additional detection layer. Another approach could be to take the average pixel information for an area of the image and apply appropriate filters to only that area. If the base of the image was dark then you could apply filters that detect lines in dark conditions there, but not to portions with light mean colors.

A further challenge was with bad fits. There were some opposing forces here. To keep the line from jumping around, I wanted to keep more frames as part of the best fit average. This would also make it feel more reasonable to allow more missed frames before having to reset and look for lanes from scratch. However, the smoother the line moves, the slower it can move when the lines are legitimately changing for a turn. As a result, I ended up using 5 frames for the average to allow more movement, and still allowing 10 frames of bad fits before resetting. With, a common 30 FPS this would give 1/3 of a second buffer, and it seemed to work well enough through the challenge video. On the other hand, the harder challenge videos lines changed so quickly that this system really struggled.

The harder challenge video also highlighted a few more problems with the approach. While my technique of placing windows based on a linear fit of pixels in the window was succesful when lines curved aggressively. These aggressive curves would move outside the bounds of my bad fit checks. As a result, the images just changed too fast for the system to handle. One way to mitigate this would be if we had access to the vehicle speed. We could then use the speed and direction to move our previous lane estimates in space for the next frame. This would give a much better estimate for detection in these fast changing cases. This could have been implemented here by guessing the speed and framerate, but I decided the guesswork was not worth the time.

The harder challenge video further caused problems with lane line detection, especially after optimizing detection for the dark section in the challenge video. Too many pixels were lit up outside of the lane lines due to the debris. As a result the lane detection via windowing and search around polynomial would get confused and follow the debris portions. Besides tuning the detection better as discussed above, we could potentially take the approach of biasing detection to the center of the lane. We could begin our detection from inside the lane for each window by some margin and proceed outwards. We have an approximation for the width of lane lines, so we could restrict the search after seeing the first N pixels + the lane width. In the case of dotted lines, you would need to make use of estimates as you go, but since the problem should mostly occur at the edge of a road, this shouldn't be a huge issue.

A final issue was that when very few pixels were detected, say one dotted lane marker, the shape of the marker could cause the fit to vary wildly if the edges were not cleanly detected. Since we know each dotted lane portion is a rectangle, we could deal with this by filling out remaining pixels before doing a line fit.
