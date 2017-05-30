# **Finding Lane Lines on the Road** 

## Writeup Template

### You can use this file as a template for your writeup if you want to submit it as a markdown file. But feel free to use some other method and submit a pdf if you prefer.

---

**Finding Lane Lines on the Road**

The goals / steps of this project are the following:
* Make a pipeline that finds lane lines on the road
* Reflect on your work in a written report


[//]: # (Image References)

[image1]: ./examples/grayscale.jpg "Grayscale"

---

### Reflection

### 1. Describe your pipeline. As part of the description, explain how you modified the draw_lines() function.

My pipeline consisted of multiple steps, at each point matplotlib was used to generate images at each step and for each test image. The resulting images can be found in the test_images_output folder.

 - Convert the image to HSL initially and apply perfrom some bitwise operations which enhances yellow and white lines
	- Previous experience with grayscale images and detection of lanes has	   thought me that some lanes depending on lighthing conditions, road surface and lane mark colour can be very difficult to even see in the raw image. Thus by viewing the image in HSL and extracting the correct range of values and re-applying to the RGB image before converting to grayscale, it aids in not only enhancing white and yellow markings but also removes many potential FPs. Note the assumption was made that only white and yellow\orange lines are targeted for this assignment.
	    
```
	def convert_hls(img):
	    """Applies the HSL transform"""
	    return cv2.cvtColor(img, cv2.COLOR_RGB2HLS)
	
	def white_color_enhance(img):
	    lower = np.uint8([  0, 200,   0])
	    upper = np.uint8([255, 255, 255])
	    white_mask = cv2.inRange(img, lower, upper)
	    return cv2.bitwise_and(img, img, mask = white_mask) , white_mask
	
	def yellow_color_enhance(img):
	    lower = np.uint8([  10, 0, 100])
	    upper = np.uint8([40, 255, 255])
	    yellow_mask = cv2.inRange(img, lower, upper)
	    return cv2.bitwise_and(img, img, mask = yellow_mask) , yellow_mask
	
	def white_yellow_enhance(img, white_mask, yellow_mask):
	    overall_mask = cv2.bitwise_or(white_mask, yellow_mask)
	    return cv2.bitwise_and(img, img, mask = overall_mask)
```	
	
- The resulting grayscale blurred image with the previous steps can be seen below:
![HSL Improvments](test_images_output/solidYellowCurve2_yellowWhiteEnhancement.JPG?raw=true)
	
 - Now that a clean grayscale image is produced (reduction in noise and
   FP potentials) the next step is to run the canny algorithm on it. And
   afterwards to create a ROI and mask everything else outside it

```
    # Canny edge detection
    low_threshold = 50
    high_threshold = 150
    canny_image = canny(blur_gray, low_threshold, high_threshold)
```

```
    imshape = canny_image.shape
    vertices = np.array([[(ROI_BOTTOM_X,imshape[0]-ROI_BOTTOM_Y),(ROI_TOP_X, ROI_TOP_Y), (imshape[1]-ROI_TOP_X, ROI_TOP_Y), (imshape[1]-ROI_BOTTOM_X,imshape[0])]], dtype=np.int32)  
    masked_canny_image = region_of_interest(canny_image, vertices)
```
	
- The resulting image is as follows:

![Canny Output](test_images_output/solidYellowCurve2_canny_maskedArea.JPG)
	

- From here the hough transform is called which in turn calls the draw_lines() which was adapted to handle line extrapolation and logic to decide where in the image the line was
  - As the output of the hough_lines() is lines we can use the slope to determine whether the lane is a left lane (negative) or right lane (positive). The slope can also be used to reject any potential FPs by ranging it between threshold values as can be seen in the following snippet
  
 ```
	slope = ((y2-y1)/(x2-x1))
            # Range the slope and remove any potential outliers
            # -ve lines are left lines
                if slope < 0: 
                    # Calculate midpoint of lines
                    left_line_x.append((x1+x2)/2)
                    left_line_y.append((y1+y2)/2)
                else:
                    right_line_x.append((x1+x2)/2)
                    right_line_y.append((y1+y2)/2)

                # Only draw segments if they meet the slope criteria
                cv2.line(img, (x1, y1), (x2, y2), color, thickness)   
```
- The following image illustrates the resulting detected road segments within the slope thresholds defined and the parameters used at the different stage. The approach I went for was having more smaller segmented lines so I have more data to fit a line through. Overall the results looks good

![Raw Lines](test_images_output/raw_line_segments.jpg)

- Afterwards the x and y lists are sorted to ensure points follow the flow of the lane. For example for a left lane; the point closest to the vehicle has image co-ordinates of (min(x) and max(y)) whilst the right lane has co-ordinates of (max(x), max(y)).

```	  
#Sort the points according to where they are expected to appear in the image
        left_line_x.sort() 
        left_line_y.sort(reverse=True) 
        right_line_x.sort()
        right_line_y.sort()
    
```

 - Now that all the points are sorted, the co-efficients and functional equation of the line can be computed. For the purpose of this assignment np.polyfit() (1st order polynomial) and np.poly1d() was used:
 	
```
 	# Calculate the co-efficients and functional equation of each line
    if len(left_line_x) > 0 and len(left_line_y) > 0:
        line_coef_left = np.polyfit(left_line_x, left_line_y, 1)
        fx_left = np.poly1d(line_coef_left)
    if len(right_line_x) > 0 and len(right_line_y) > 0:
        line_coef_right = np.polyfit(right_line_x, right_line_y, 1)
        fx_right = np.poly1d(line_coef_right)
```

- cv2.line() was used to fit the final line through the f(x) equations

```
# Fit the lines
    if len(left_line_x) > 0 and len(fx_left) > 0:
        cv2.line(img, (int(left_line_x[-1]), int(fx_left(int(left_line_x[-1])))), (ROI_BOTTOM_X, int(fx_left(ROI_BOTTOM_X))), color, thickness)
    if len(right_line_x) > 0 and len(fx_right) > 0:
        cv2.line(img, (int(right_line_x[0]), int(fx_right(int(right_line_x[0])))), (img.shape[1]-ROI_BOTTOM_X, int(fx_right(img.shape[1]-ROI_BOTTOM_X))), color, thickness) 
```

- The final resultant image and videos with the fitted overlays (in blue) and also the segmented results can be found in the following image and videos. As it can be seen, the fit is accurate for this data. In the next sections I will highlight the weaknesses and potential improvements.

![Final Lines](test_images_output/final_line_outputs.jpg)

**Solid White Right Result**
![Final Lines Video 1](test_videos_output/solidWhiteRight.gif)



### 2. Identify potential shortcomings with your current pipeline


The pipeline has the following shortcomings:
- In curved road scenes the first order model will not hold well and the resulting detection would be impacted accordingly.
- Lines in other colours than white or yellow\orange may not be detected well. For example blue lines may be difficult to detect.
- Lane changing scenarios may not be handled well as the lanes will be in the center of the image thus they might be detected well or rejected due to slope.
- Nearby vehicles be it travelling in the same direction or in the opposite direction may cause FPs or affect the overall detection


### 3. Suggest possible improvements to your pipeline

The following highlights some key areas which could be improved upon:
- The line fitting could possibly be improved using other libs\functions available or by writing my own
- Keeping a history of the slopes and lines so that weighting could be applied to the current frame base on the previous frame(s)
- The length of the line and its history could also in preventing any major jumps in the fitted line which may suggest FP detections


