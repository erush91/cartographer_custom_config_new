*** 2019-10-14 ***

SENSORS

1 x Horizontal OS-1-64, 2048x10 mode 	GOOD	
1 x Vertical OS-1-64, 2048x10 mode 	GOOD	
1 x Horizontal RPLIDAR S1 		BAD	laser frame, TOO MANY MESSAGES	
1 x Vertical RPLIDAR S1 		BAD	laser frame,

SENSOR COMBINATIONS

Single Horizontal Ouster testing is okay, but is at 2 x resoluation compared to Tunnel Circuit
Running Horizontal and Vertical Ouster does not work, issue with Cartographer integration (the scan_matched_points alternate between the two sensors)
Same issue for Horizontal Ouster and Vertical RPLIDAR.
Cannot do any 2D LiDAR testing because the horizontal LiDAR is publishing too often (appears blinking in RViz)

TUNING PARAMETERS

*** 2019-10-15 ***

SENSORS

1 x Horizontal OS-1-64, 1024x10 mode 	GOOD
1 x Vertical OS-1-64, 1024x10 mode	GOOD
1 x Horizontal RPLIDAR S1		GOOD	rp0 frame
1 x Vertical RPLIDAR S1			GOOD	rp1 frame

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
- what the proper IMU calibration / urdf is...
- 


