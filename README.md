# Udacity SDCND - Advanced Lane Finding

## 1. Generalities

Please also see the LaneFinder.ipynb file, as I worked there for the most part. I also committed the HTML
version of it to GitHub. Project output video is simply output.mp4

This project ended up being far more tedious than I expected. In particular, it is difficult to distinguish
the lines on some 'whiter' section of the road, though shadows themselves are not much of a problem.

The approach used is pretty much what was described in the lectures, with a few caveats

		1.- I used a clustering algorithm (DBSCAN, and then one of mine) to try to provide, first, a rough 
			classification of the lanes in each image. This is done by slicing the warped binary image
			that identifies the lane markes, applying DBSCAN in each slice, and then, using those found
			cluster, applying another clusterin algo to try to identify the lanes, by moving through the
			points in an ascending fashion. This seems more elegant than just looking for the peaks in the
			histogram, though far, FAR more expensive. But I just could not do that, it was ugly AND clearly
			failure prune. Also I wanted to get ALL lines possible, if there are more visible and recognized.
		2.- I just cannot do well enough on those white sections of the road. It is just frustrating. The
			pipeline does not dies of course, but lane identification is clearly subpar
		3.- I applied an EM (Expectation Maximization of sorts), which is just saying that the found lanes
			in (1) are then used to get preliminary fits, and the these are used as starting points against
			the warped lane markers binary image to try to get better fits using a few iterations, defining
			a 'range' around the fit to classify valid points. Clearly, a robust fittig algo could work
			better but I had little time for this.
		4.- The fits themselves are used to get the curvatures, and reported curvature is an average of the
			side curvatures. An EMA filter is used to try to filter these, but it fails on the white patches
			(in the way that it on itself does not prunes outliers)
		5.- There is no sophisticated outlier pruning algo, I wanted to do something like 
			http://acds-lab.gatech.edu/papers/IROS2007_RobustKF.pdf though. I will do so in the future, but not
			now nor for this project.
			
So, in the detail, the pipeline does this:

		0.- Get calibration matrix using images, this is done in the first sections of the IPYTHON notebook
		1.- Undistort the image and perform some mild preprocessing on it (channel normalization)
		2.- Apply various masks (sobel, ranges, in different color spaces) over the image to try to
			isolate the points of the lane markers and produce this binary lane markers image with the data
			from the different mask applied to the image and some logical conditions. By FAR this was the more
			annoying part, as it is not easy to select good parameters so tuning was slow and error prone.
			I strongly suggest using a DNN instead, applying a convolution over the image that returns a 
			single bit (lane marker or else) and work over that. For efficiency the lane markers binary image
			used is scaled
		2.- Warp the image to fix perspective, the parameters were hand tuned after numerous trials
		3.- Slice the image, and obtain clusters of the lane marker points.
		4.- Use another clustering algo over those points to try to identify the lane marker lines. This is
			important since a lane marker could be segmented into discontinuos chunks, so it was an interesting
			problem. The approach was walking the list of points in ascendir order of Y, and checking distances
			in a 'stretcehd' space of the original points, and finding best matchs within a tolerance, else
			assigning a new label/class, BUT using only the 'topmost' (in Y) point of each 'class'. This works
			nicely but endless variations are possible. Generalized Houghs seemed too expensive
		5.- Nevertheless, the 'lines' found in (4) are of low quality, so I get a 2-deg poly fit of those, and
			then (and various checks) use those fits as starting points to obtain better fits over the lane markers
			image after a few iterations.
		6.- These fits are classified on side, and the closest to the car are used to determine side curvatures
			and lateral distances.
		7.- Filtering is very weak, just an EMA algo. For outlier detection, actually the fits classificator
			could fail, and if that happens (because the fit was 'bad' or there were no points) the last 'good'
			fit is used. The tolerance of the classificator could be reduced to better reject outliers. The paper
			referenced before though seems the ideal for a more robust solution, but it is complex to implement 
		8.- The fits found are used to generate a green area that is transformed back to the original image ( though
			undistorted ), and the curvature data is averaged to get the lane curvature, and the lateral offset
			is calculated with the fits
		
## 2. Future work

		1.- Clearly use the robust Kalman filter for filtering everything at the 'front end' stage, as values can 
			change quite wildly otherwise. A more sophisticated outlier rejection, even if not a robust KF, could 
			also work, for example, getting the R**2 of the fits and using a threshold over it.
		2.- The whiter patches are hard. I strongly think that even if I managed to correctly process them, that 
			will make the 'model' brittle due to fine tuned parameters, and will fail in other cases.
		3.- It will fail in the challenges, though I could make it work in the easier one, at the cost of reducing
			performance on the project video (as said in (2)). In particular, it will fail on low illumination or
			those bad white patches.
		4.- The harder challenge requires far more sophisticated ideas. Using splines seems a must. Also, fitting
			first on the unwarped image and then warping the fits seems that could work better. A depth sensor
			could significantly improve performance. Does Tesla manages to do this well? Without maps? I would
			be surprised, since they dont usar LIDAR.
		5.- Tuning, tuning, tuning.
		6.- More sophisticated preprocessing of the images could help. Gradient equalization, CLAHE, DNNs could
			work MUCH better after very extensive training.
			
## 3. Caveats

Rubric:
	
			In the first few frames of video, the algorithm should perform a search without prior assumptions about 
			where the lines are (i.e., no hard coded values to start with). Once a high-confidence detection is 
			achieved, that positional knowledge may be used in future iterations as a starting point to find the lines.
			
Is not strictly followed, as the search is done over ALL frames. Clearly this code is not good for production, but
perhaps FPGAs could do it, after serious optimization.

Rubric:

			As soon as a high confidence detection of the lane lines has been achieved, that information should be 
			propagated to the detection step for the next frame of the video, both as a means of saving time on 
			detection and in order to reject outliers (anomalous detections).

Is realized in a way, on each posterior frame, the previous best fits are added to the list of fits for optimization.
This helps for outliers, as if a fit is the best, it will be selected over the others (for each side). Expensive though,
as I actually consume more time for doing this, so I do not save any time.