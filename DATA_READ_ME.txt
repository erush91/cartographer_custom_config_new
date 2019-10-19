*** 2010-10-09 *** 

--------------------

DATA COLLECTION COMPLETED

SETUP: F450 UAV

SENSORS:

1 x Horizontal OS-1-64, 1024x10 mode    GOOD

ENVIRONMENT: Aerospace Flight Arena, Mike O Manual Flight, difficult to take off, control, and fly for long. Short tests

--------------------

*** 2019-10-14 ***

--------------------

DATA COLLECTION: 

SETUP: FULL HANDHELD RIG (OVERHEAD GENE)

SENSORS:

1 x Horizontal OS-1-64, 2048x10 mode 	GOOD	
1 x Vertical OS-1-64, 2048x10 mode 	GOOD	
1 x Horizontal RPLIDAR S1 		BAD	laser frame, TOO MANY MESSAGES	
1 x Vertical RPLIDAR S1 		BAD	laser frame,

ENVIRONMENT: 1st Floor and Basement ECES

--------------------

SENSOR COMBINATIONS NOTE

Single Horizontal Ouster testing is okay, but is at 2 x resoluation compared to Tunnel Circuit
Running Horizontal and Vertical Ouster does not work, issue with Cartographer integration (the scan_matched_points alternate between the two sensors)
Same issue for Horizontal Ouster and Vertical RPLIDAR.
Cannot do any 2D LiDAR testing because the horizontal LiDAR is publishing too often (appears blinking in RViz)

*** 2019-10-15 ***

--------------------

DATA COLLECTION COMPLETED

SETUP: FULL HANDHELD RIG (OVERHEAD GENE)

SENSORS:

1 x Horizontal OS-1-64, 1024x10 mode 	GOOD
1 x Vertical OS-1-64, 1024x10 mode	GOOD
1 x Horizontal RPLIDAR S1		GOOD	rp0 frame
1 x Vertical RPLIDAR S1			GOOD	rp1 frame

ENVIRONMENT: 1st Floor and Basement ECES

--------------------

TUNING 

Reverted back to tuning parameters from H02_3d.lua from before Tunnel Circuit. This is when we were using 3D Cartographer, before we made the switch to 2D Cartographer. Comparison has not been done between thesethese parameters and the "tuned" parameters from 2019-10-14.

URDF

Implemented Mike M's tf tree, resulted in worse results. The drift was so large that the loop closure latency grew, causing unrepairable drift. Even after sitting for an hour, did not loop close. NEED TO REVISIT, I BELIEVE THIS DIDN'T WORK DUE TO AN ERROR ON MY END. WOULD BE BETTER TO HAVE IN TRACKING FRAME.

Reverted to my tf tree, with Mike M's CAD sensor origins, worked well.

Analyzed IMU correction: Adding this correction DID reduce  drift. This 
- reduced pitch by 0.33 deg (cartographer_tuning_10rot_10trans_minus0pt33_yaw_2019-10-15-22-49-21.bag)
- increased roll  by 0.15 deg

Running OL (without loop closures) achieved less drift than CL with IMU extrinics calibration running lone (true). Surprising
Ran again CL (with loop closures) and turned off automatic IMU extrinsics, found that it had no effect, it is still worse than running OL. Maybe this is because the Local SLAM has less resources and is fighting the Global SLAM when Global SLAM is turned on? 

TRAJECTORY_BUILDER_3D.ceres_scan_matcher.rotation_weight:TRAJECTORY_BUILDER_3D.ceres_scan_matcher.translation_weight

Originally was using 10:1. Found this had low drift but some noisy trajectories. Using 10:10 achieved even lower drift, and smoothed out trajectories.

ROTATIONAL:TRANSLATIONAL = 1:1 rough trajectories (bad), lower drift (really good)
ROTATIONAL:TRANSLATIONAL = 10:1 rough trajectories (bad), low drift (good)
ROTATIONAL:TRANSLATIONAL = 10:10 smooth trajectory (good), lowest drift (really really good)
ROTATIONAL:TRANSLATIONAL = 100:100 really smooth trajectory (really good), slips in lobby (bad), more drift (bad)
ROTATIONAL:TRANSLATIONAL = 300:100 really smooth trajectory (really good), slips in lobby (bad), more drift (bad)
ROTATIONAL:TRANSLATIONAL = 1000:100 really smooth trajectory (really good), slips in lobby (bad), more drift (bad)

cartographer_tuning_10rot_10trans_plus0pt15_roll_minus0pt33_yaw_fixedZ2019-10-15-22-49-21.bag

With 10:1 and modified IMU tf based on IMU correction, getting better OL results. When I try CL, it seems like loop closures are not occuring, and I do't know why. The reason was because I accidentally had POSE_GRAPH.optimization_problem.fix_z_in_3d = true, when it should have been false. 

cartographer_tuning_2019-10-15-23-46-24.bag

Now loop closures are happening. But, the drift is greater now than when it was running open loop. Loop closures happen, but with a normal amount of latency.

cartographer_tuning_2019-10-16-00-05-33.bag

Now trying to make the pose graph ceres weights match the trajectory bui;der 3d, changed rotation_weight from 100 to 10. Actually has more drift and no loop closures for 10.

Put back to 10 and put the traj trans back to 10 too: cartographer_tuning_2019-10-16-00-18-18.bag. Still suffering slow loop closures.

Reduced # nodes to from 320 to 90, still large latency for loop closures: cartographer_tuning_2019-10-16-00-32-16.bag

Decreased global_sampling_ratio and constraint_builder.sampling_ratio: cartographer_tuning_2019-10-16-00-46-00.bag

Loop closures are still slow. Reduced POSE_GRAPH.optimization_problem.ceres_solver_options.max_num_iterations from 10 to 5 and POSE_GRAPH.constraint_builder.min_score from 0.62 to 0.55: cartographer_tuning_2019-10-16-01-19-25.bag --> FASTER LOOP CLOSURES!!!

Now accumulated drift is lower, can reduce POSE_GRAPH.constraint_builder.fast_correlative_scan_matcher_3d.linear_xy_search_window from 10 to 5 and POSE_GRAPH.constraint_builder.fast_correlative_scan_matcher_3d.linear_z_search_window from 5 to 3 m: EVEN FASTER LOOP CLOSURES!!! ALMOST REAL TIME!!! cartographer_tuning_2019-10-16-01-19-25.bag

rqt_plot shows that the maximum angular velocities are around 0.75 rad/s. Should use this information in future parameter tuning


Did some more reading about the IMU calibration: it says the output is the lidar sensor to imu frame: https://github.com/googlecartographer/cartographer_ros/pull/1025

Unknowns:
- what the proper IMU calibration / urdf is...should try adding 5 deg error to mimic large error, then look at extrinsics report.
- getting smoother trajectory (lowering traj weights) without causing slipping.

Ran 2048x10 (cartographer_tuning_2019-10-16-02-40-03.bag) to compare vs the 1048 x 10: It performs decently, the amount of drift is similar (both low). But the loop closures have more delay (greater latency), which is expected due to having 2x as many data points to optimize over. I recommend using 1024x10 over 2048x10 for the aerial vehicle, since it has limited compute.

try setting translation back to 10, ran overnight...

setting POSE_GRAPH.constraint_builder.ceres_scan_matcher_3d.rotation_weight = 10 is similar to POSE_GRAPH.constraint_builder.ceres_scan_matcher_3d.rotation_weight = 100. ACTUALLY, setting to 10 and running overnight resulted in a bad loop closure of the basement and 1st floor.

*** 2019-10-16 ***

Tested on 2019_10_09_F450_Flight_Aero_test1, test2, test3 dataset, looks really good, EXCEPT the bad loop closure at the beginning. Need to figure out how to prevent that from happening.

Also some linear error, I think because the vehicle is moving fast.

Tried to change false -- true: POSE_GRAPH.constraint_builder.ceres_scan_matcher_3d.optimize_yaw = false. It did not help at all. I think this issue may be at the traj level, not pose graph.

Tried traj ang limit from math.rad(1) --> math.rad(0.001) did not fix either, so not traj. TRAJECTORY_BUILDER_3D.real_time_correlative_scan_matcher.angular_search_window

Now tried to change pose graph from math.rad(22.5) to math.rad(1): POSE_GRAPH.constraint_builder.fast_correlative_scan_matcher_3d.angular_search_window = math.rad(1)...This also didn't help

FIGURED IT OUT: Had to set POSE_GRAPH.optimization_problem.use_online_imu_extrinsics_in_3d = false. It was true before, and wreaking havoc!

Now need to retest without online imu extrinsics on ECES dataset.

cartographer_tuning_2019-10-16-13-42-11.bag

Worked well without online IMU extrinsics.

*** 2019-10-17 ***

I tried to add some IMU orientation error but the results are inconsistent. I added 1 deg of error, and it couldn't output that. I tried assuming it is w, x, y, z. I also tried x, y, z, w.

To Do Items
- faster loop closures, so accumulate less drift
- less noisy trajectories
- should try single scan Ouster
- run with less than 6 cores (600-700%)

Now I don't trust the IMU extrinsics estimation output. I am going to just TUNE EXTRINSICS BY HAND! RUNNING WITHOUT LOOP CLOSURE



ALSO, NOW LOOKING AT 2 x 2D LIDARS

Can't run 2 x 2D LiDARS. For some reason, it goes off to infinity. I had the same issue with the Ouster.

Running 3D Cartographer with 1 Ouster (horizontal) is not great. Significant drift, much of which is caused by bad scan matches or "jumps" due to quick rolling or pitching motion. This causes the sensor to see the ground instead of the walls. A limitation of using a 2D sensor to do 3D state estimation. Also, the accumulated z drift is pretty bad.

Increasing traj rot and transl weights by 10x helped. Should probably try to continue increasing and see what happens. May be usable for control, but still a "jump" here or there.

To Do Items
- increase traj rot and transl weights to see what happens.
- try to reduce upward z drift 
- try to get more loop closures. Difficult to tell if it was even doing anything.

POSE_GRAPH.constraint_builder.ceres_scan_matcher_3d.only_optimize_yaw = false --> true (didn't really do much)

POSE_GRAPH.optimization_problem.fix_z_in_3d = false --> true (didn't fix z, but reduced the z translation by a lot, so not accurate, not what we want.

only uses 30-60% compute! Seemed like it wasn't doing many loop closures

*** 2019-10-18 ***

Trying to reduce drift in open loop

Tuning the local SLAM right now, and just had another breakthrough. These parameters have a great effect on the total odometry drift. I just ran without loop closures and this is my map. I've never gotten results this good before.
TRAJECTORY_BUILDER_3D.motion_filter.max_time_seconds = 5
-- TRAJECTORY_BUILDER_3D.motion_filter.max_time_seconds = 0.5
TRAJECTORY_BUILDER_3D.motion_filter.max_distance_meters= 1
-- TRAJECTORY_BUILDER_3D.motion_filter.max_distance_meters= 0.1
TRAJECTORY_BUILDER_3D.motion_filter.max_angle_radians = 0.4
-- TRAJECTORY_BUILDER_3D.motion_filter.max_angle_radians = 0.004

What this is doing is simply using less of the lidar scans. Which I guess makes it more accurate...?

now need to try closed loop

Tried closing loop, found that loop closures aren't happened (as frequently as I would like), maybe I need to tune the optimize every n nodes, since maybe there's less nodes now that I am sampling so infrequently? Maybe I should be less aggresive with the motion filter.


