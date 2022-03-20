## Introduction
The purpose of this project is to create a lane line detection program as a driving-assistant feature in a vehicle. Drivers assistant features like this have become increasingly widespread on new car models. Toyota released cars with the first features like this in 2004 (North America). Currently, Chevrolet advertises their “Lane Keep Assist” and “Lane Departure Warning” systems, Ford has a similar technology as a part of “Ford Co-Pilot360”, and many other car companies feature equivalent systems in their cars. Technologies such as this are an intermediate stepping-stone on the road to full autonomous driving and, if implemented well, may encourage a positive public perception for the fully autonomous cars of the future. 

My project focuses on building a prototype of a lane-assist feature and analyzing the real-time performance of the computer vision algorithm, and the accuracy of the algorithm using the receiver operating characteristic (ROC). I also demonstrate certain video inputs where the algorithm performance is poor and propose some areas for further study and development in order to correct these issues. Since I had limited time, I opted to use a set of pre-recorded challenge set videos (courtesy of Sam Siewert) to test and evaluate my prototype. 

## Functional Design
The functional design relies upon an application written in C++ with OpenCV 3 (version 3.4.6). The application uses pthreads for symmetric multiprocessing (SMP). The system could easily be modified for asymmetric multiprocessing (AMP) by setting the core affinities of each of the respective threads. [Figure 2](figures/Fig2.png) illustrates the separation of functionalities for each of the threads, and the transmission of data between them. 

<p align="center">
  <img src="figures/Fig2.png" width="400" title="Figure 2">
</p>

The design relies upon single producer single consumer lock-free circular buffers. I did not spend time writing the ring buffer implementation from scratch and instead opted to use a freely available source code from [Dennis Lang](https://landenlabs.com/code/ring/ring.htm) for the lock-free ring buffers. 
The bulk of the computer vision processing occurs inside the LaneDetector class, defined in the lane.cpp/h files. The final prototype uses the pipeline shown in [Figure 3](figures/Fig3.png). 

<p align="center">
  <img src="figures/Fig3.png" width="600" title="Figure 3">
</p>

The individual steps in the processing pipeline are discussed in further detail below. 

#### Region of Interest (ROI) Extraction
The raw BGR frame used as input is converted to grayscale. The ROI is then cropped out of the grayscale frame using a predetermined static rectangle of four fixed points which define the region of interest (ROI). I also apply a median filter to the grayscale ROI after cropping to reduce noise. In a professional solution, this would likely depend on the camera placement and orientation and may even contain greater than four points. The predetermined ROI for clip1.avi is shown in [Figure 4](figures/Fig4.png). 

<p align="center">
  <img src="figures/Fig4.png" width="500" title="Figure 4">
</p>

#### Binary Adaptive Threshold
A 5x5 kernel mean adaptive threshold is applied to the grayscale ROI resulting in the binary ROI shown in [Figure 5](figures/Fig5.png). 

<p align="center">
  <img src="figures/Fig5.png" width="500" title="Figure 5">
</p>

The binary ROI is ready for line detection using the Hough transform. The Hough transform is relies on a transformation from x, y image coordinates to ⍴, θ space. In this transformed coordinate system, it is simpler to find the number of pixels that are along a line. The OpenCV Hough function allows one to specify the minimum and maximum θ to search between and returns a set of ⍴, θ lines which have accumulated votes (white pixels along that line) greater than a threshold. [Figure 6](figures/Fig6.png) shows the convention that OpenCV uses for ⍴ and θ. The top left corner is the (0,0) point of the binary ROI. 

<p align="center">
  <img src="figures/Fig6.png" width="500" title="Figure 6">
</p>

The Hough transform returns all the lines that are above a threshold but [Figure 6](figures/Fig6.png) illustrates the idea that we can expect the left and right lane lines are within certain bounds in ⍴, θ space which we can leverage to filter the Hough results.  If, for example, no left lane line is found after applying the filter then this line is omitted from the annotation process in the next stage of the pipeline.  We can then use these lane lines to estimate the offset distance between a predefined center point and for the lane departure warning. 


#### Generalized Annotation
The annotation process serves to present all of this information as an annotation overlay upon the raw frame. [Figure 7](figures/Fig7.png) shows an example of the raw image after annotation. The ROI is the thin blue bounding box, the left and right lane lines are annotated in red, the red tick mark is the center (calculated using left and right lines), and finally the green tick mark is the predefined centerline for the vehicle. The green tick mark will change color depending on the offset from the red tick mark and could appear green, yellow, or dark red for increasing offsets from center.  In [Figure 7](figures/Fig7.png), the car is drifting to the right. 
<p align="center">
  <img src="figures/Fig7.png" width="500" title="Figure 7">
</p>


## Performance Analysis
#### Real-time analysis
The real-time analysis consisted of frame rate throughput calculations as well as an evaluation of jitter. For determining these statistics, the code relies on the logging module supplied in the log.cpp/h files. This includes the macros LOGP and LOGSYS for logging to printf and syslog, respectively. This module also provides a wrapper for the linux function clock_gettime with CLOCK_MONOTONIC to return the time in milliseconds (ms). 

The frame rate is calculated as the number of frames processed per second (FPS) during execution of the program. One thing to note is that for my setup, processing throughput on the Jetson is seriously hindered when displaying remote X graphics over SSH (running the program with the --show=1 option). Therefore, the real-time performance of the system is based on when executing with --show=0 (default). Upon program interrupt or completion, FPS and other metrics will be logged to the console.

One of the other considerations for a system like this is the real-time jitter (variation between the expected timing and the actual timing for a task). I used a syslog statement (with a known tag) after each frame annotation for real-time jitter analysis.  The command `$tail -n 10000 /var/log/syslog | grep @CV >> timestamps.txt` was used to extract the proper timestamps. These were then imported into excel for calculation and plotting. [Figure 9](figures/Fig9.png) shows the time period difference in ms between frame annotations. 

<p align="center">
  <img src="figures/Fig9.png" width="500" title="Figure 9">
</p>

The blue line is the time difference between subsequent frame annotations which demonstrates the jitter in frame processing. The yellow line is a 100 point moving average of the blue line which remains relatively stable over time and is not trending at an upward or downward slope. 

#### Computer vision accuracy ROC
For lane detection ROC analysis, I determined the number of true positives, true negatives, false positives, and false negatives in terms of lane line detections. For simplicity, I constrained each frame to a maximum of two possible lane lines (left and right). The definitions for these parameters are listed below: 
- True positive - the program identifies a lane line and a lane line exists in that region of the frame
- True negative - the program does not identify a lane line and there is not a lane line in that region of the frame
- False positive - the program identifies a lane line but there is no lane line in that region of the frame
- False negative - the program does not identify a lane line and a lane line exists in that region of the frame
	
The ROC of [Figure 10](figures/Fig10.png) is a plot of the true positive rate (TPR), or sensitivity vs. the false positive rate (FPR), or 1-specificity as a function of some parameter. In reality, this process should be automated. A more robust analysis method would have a dataset of images with human annotated lane lines and then compare the algorithm output with the human-derived output for generating the ROC curves for various parameter settings automatically. My hand analysis will focus on a set of 1200 frames from clip1.avi and clip2.avi at various Hough accumulator threshold settings, but the ideas presented could be extended to an automated analysis scheme in the future. 
<p align="center">
  <img src="figures/Fig10.png" width="500" title="Figure 10">
</p>

You can see that the system exceeds the minimum TPR and FPR goal when the threshold is at 50. It also exceeds stretch and target goals for FPR, but not for TPR. 
This analysis leads to the conclusion that we would need other detection methods in addition to the lane detection program shown in order to have a robust and trustworthy system. One method would be to add a stoplight or intersection detection feature that would disable lane detection when going through intersections so that the system above would not constantly be annoying the driver as they drove through intersections. 


## Conclusion
The purpose of this project was to create a lane line detection program as a driving-assistant feature in a vehicle. In this report I have focused on the design of my prototype lane-assist feature and have provided analysis in terms of the real-time performance and the accuracy using the receiver operating characteristic (ROC). Overall, the project meets the requirements in terms of real-time performance; however, could likely be improved in terms of accuracy. The false positive and false negative rates are likely too great to rely on this algorithm alone for a commercial grade lane detection system.  The solution presented would need to be coupled with other subsystems in order to be a useful and trustworthy lane detection system. 

There are many potential improvements to this application. First, investigating changes to the image processing pipeline such as using Canny/Sobel for extracting the binary ROI, an improved heuristic for Hough line filtering, using histogram based methods instead of Hough, or perspective warping could result in greater algorithm accuracy. In addition, taking the design further to detect more than two lane lines and lane curvature could be an interesting exercise. Transferring the code to asymmetric multiprocessing and SCHED_FIFO may yield better real-time performance and reduce jitter. Finally, integration of this into an actual commercial application would require live camera integration and calibration, evaluation of real-time system deadlines, and sensor fusion with multiple cameras, IMU, GPS or IR sensors. 
