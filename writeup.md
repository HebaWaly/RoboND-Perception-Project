## Project: Perception Pick & Place
### Writeup Template: You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

---


# Required Steps for a Passing Submission:
1. Extract features and train an SVM model on new objects (see `pick_list_*.yaml` in `/pr2_robot/config/` for the list of models you'll be trying to identify). 
2. Write a ROS node and subscribe to `/pr2/world/points` topic. This topic contains noisy point cloud data that you must work with.
3. Use filtering and RANSAC plane fitting to isolate the objects of interest from the rest of the scene.
4. Apply Euclidean clustering to create separate clusters for individual items.
5. Perform object recognition on these objects and assign them labels (markers in RViz).
6. Calculate the centroid (average in x, y and z) of the set of points belonging to that each object.
7. Create ROS messages containing the details of each object (name, pick_pose, etc.) and write these messages out to `.yaml` files, one for each of the 3 scenarios (`test1-3.world` in `/pr2_robot/worlds/`).  See the example `output.yaml` for details on what the output should look like.  
8. Submit a link to your GitHub repo for the project or the Python code for your perception pipeline and your output `.yaml` files (3 `.yaml` files, one for each test world).  You must have correctly identified 100% of objects from `pick_list_1.yaml` for `test1.world`, 80% of items from `pick_list_2.yaml` for `test2.world` and 75% of items from `pick_list_3.yaml` in `test3.world`.
9. Congratulations!  Your Done!

# Extra Challenges: Complete the Pick & Place
7. To create a collision map, publish a point cloud to the `/pr2/3d_map/points` topic and make sure you change the `point_cloud_topic` to `/pr2/3d_map/points` in `sensors.yaml` in the `/pr2_robot/config/` directory. This topic is read by Moveit!, which uses this point cloud input to generate a collision map, allowing the robot to plan its trajectory.  Keep in mind that later when you go to pick up an object, you must first remove it from this point cloud so it is removed from the collision map!
8. Rotate the robot to generate collision map of table sides. This can be accomplished by publishing joint angle value(in radians) to `/pr2/world_joint_controller/command`
9. Rotate the robot back to its original state.
10. Create a ROS Client for the “pick_place_routine” rosservice.  In the required steps above, you already created the messages you need to use this service. Checkout the [PickPlace.srv](https://github.com/udacity/RoboND-Perception-Project/tree/master/pr2_robot/srv) file to find out what arguments you must pass to this service.
11. If everything was done correctly, when you pass the appropriate messages to the `pick_place_routine` service, the selected arm will perform pick and place operation and display trajectory in the RViz window
12. Place all the objects from your pick list in their respective dropoff box and you have completed the challenge!
13. Looking for a bigger challenge?  Load up the `challenge.world` scenario and see if you can get your perception pipeline working there!

[//]: # (Image References)

[objects]: ./writeup_images/objects.png
[table]: ./writeup_images/table.png
[clusters]: ./writeup_images/clusters.png
[confusion1]: ./writeup_images/confusion1.png
[confusion2]: ./writeup_images/confusion2.png
[confusion3]: ./writeup_images/confusion3.png
[world1]: ./writeup_images/world1.png
[world2]: ./writeup_images/world2.png
[world3]: ./writeup_images/world3.png

## [Rubric](https://review.udacity.com/#!/rubrics/1067/view) Points
### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  

You're reading it!

### Exercise 1, 2 and 3 pipeline implemented
#### 1. Complete Exercise 1 steps. Pipeline for filtering and RANSAC plane fitting implemented.
First of all I copied the given `project_template.py` file to a new one named `object_recognition.py` and this is where I implemented the pipeline.
Then I applied the following filters in order:
1. Statistical outlier, to remove the noise.
2. Voxel grid downsampling using leaf size of 0.01, to increase the processing speed.
3. PathThrough filter:
  first along Z-axis from 0.6 to 1.1, to get rid of whatever is below the table top, which I don't need to worry about.
  Second along Y-axis from -0.5 to 0.5, to crop the dropboxes from the image, so that the algorithm won't try to classify them as objects.
After that I applied RANSAC plane segmentation to separate the table top from the objects, each in a separate cloud, using distance threshold of 0.0175.
![objects cloud][objects]
![tabletop cloud][table] 
#### 2. Complete Exercise 2 steps: Pipeline including clustering for segmentation implemented.  
The next step was to use Euclidean Clustering on the object cloud, using cluster tolerence 0.02, minimum cluster size 20 and maximum 2500. The result looks as follows:
![cluster cloud][clusters]
#### 3. Complete Exercise 3 Steps.  Features extracted and SVM trained. Object recognition implemented.
Next I got to extract the features from the models using scripts from the `sensor_stick` module. But in order to raise the accuracy of my model, I had to make some changes to it:
1- In `sensor_stick/src/sensor_stick/features.py` [here](https://github.com/HebaWaly/RoboND-Perception-Exercises/blob/master/Exercise-3/sensor_stick/src/sensor_stick/features.py) I changed the number of bins used to construct both the color and normals histograms from 32 to 200.
2- In `capture_features.py` I changed the loop do a 100 sample per object instead of 5.
3- I used hsv when computing the coloured histograms.
Those 3 changes helped me improve my confusion matrix a lot as shown in the figure below:
![confusion matrix 3][confusion3]
### Pick and Place Setup

#### 1. For all three tabletop setups (`test*.world`), perform object recognition, then read in respective pick list (`pick_list_*.yaml`). Next construct the messages that would comprise a valid `PickPlace` request output them to `.yaml` format.
After training my model, the algorithm was able to recognize all objects in the 3 worlds correctly.
![world1][world1]
![world2][world2]
![world3][world3]

The most challenging part was increasing the accuracy of the model such that it could recognise all the objects in all the 3 test cases.
First, I used hsv and increased the data samples taken in [capture_features.py](https://github.com/HebaWaly/RoboND-Perception-Exercises/blob/master/Exercise-3/sensor_stick/src/sensor_stick/features.py) script to 50, and this was enough to pass test #1, but wasn't enough to pass test #2.
![confusion1][confusion1]
So I changed the bins number of the histograms from 32 to 64 in both `compute_color_histograms()` and `compute_normal_histograms()` methods, and this helped me raise my confusion matrix for test #2 and identify all 5 objects, but again this wasn't enough for test #3 to work.
![confusion2][confusion2]
So then I raised the bins number again from 64 to 200, and raised the samples loop counter from 50 to 100 in [capture_features.py](https://github.com/HebaWaly/RoboND-Perception-Exercises/blob/master/Exercise-3/sensor_stick/src/sensor_stick/features.py) and my third confusion matrix was good to proceed with.
![confusion3][confusion3]

All output files are saved in the `/scripts` directory.


