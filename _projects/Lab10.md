---
layout: project
title: Lab 10
description: Lab 10 Writeup
permalink: /lab10/
---

## Functions

<pre><code class="language-python">def compute_control(cur_pose, prev_pose):
    """ Given the current and previous odometry poses, this function extracts
    the control information based on the odometry motion model.

    Args:
        cur_pose  ([Pose]): Current Pose
        prev_pose ([Pose]): Previous Pose 

    Returns:
        [delta_rot_1]: Rotation 1  (degrees)
        [delta_trans]: Translation (meters)
        [delta_rot_2]: Rotation 2  (degrees)
    """

    delta_rot_1 = mapper.normalize_angle(np.degrees(np.arctan2(np.radians(cur_pose[1] - prev_pose[1]),np.radians(cur_pose[0]-prev_pose[0]))-np.radians(cur_pose[2])))
    delta_trans = np.sqrt((cur_pose[0] - prev_pose[0])**2 + (cur_pose[1] - prev_pose[1])**2)
    delta_rot_2 = mapper.normalize_angle(cur_pose[2] - prev_pose[2] - delta_rot_1)

    return delta_rot_1, delta_trans, delta_rot_2
</code></pre>

<pre><code class="language-python">def odom_motion_model(cur_pose, prev_pose, u):
    """ Odometry Motion Model

    Args:
        cur_pose  ([Pose]): Current Pose
        prev_pose ([Pose]): Previous Pose
        (rot1, trans, rot2) (float, float, float): A tuple with control data in the format 
                                                   format (rot1, trans, rot2) with units (degrees, meters, degrees)


    Returns:
        prob [float]: Probability p(x'|x, u)
    """

    delta_rot_1, delta_trans, delta_rot_2 = compute_control(cur_pose, prev_pose)

    prob_rot_1 = loc.gaussian(delta_rot_1, u[0], loc.odom_rot_sigma)
    prob_trans = loc.gaussian(delta_trans, u[1], loc.odom_trans_sigma)
    prob_rot_2 = loc.gaussian(delta_rot_2, u[2], loc.odom_rot_sigma)

    prob = prob_rot_1 * prob_trans * prob_rot_2
    
    return prob
</code></pre>

<pre><code class="language-python">def prediction_step(cur_odom, prev_odom):
    """ Prediction step of the Bayes Filter.
    Update the probabilities in loc.bel_bar based on loc.bel from the previous time step and the odometry motion model.

    Args:
        cur_odom  ([Pose]): Current Pose
        prev_odom ([Pose]): Previous Pose
    """

    u = compute_control(cur_odom, prev_odom)
    for x_prev in range(mapper.MAX_CELLS_X):
        for y_prev in range(mapper.MAX_CELLS_Y):
            for th_prev in range(mapper.MAX_CELLS_A):
                if(loc.bel_bar[x_prev,y_prev,th_prev] < 0.0001): # skip loop if belief too low
                    continue
                for x in range(mapper.MAX_CELLS_X):
                    for y in range(mapper.MAX_CELLS_Y):
                        for th in range(mapper.MAX_CELLS_A):
                            prev_pose = mapper.from_map(x_prev, y_prev, th_prev)
                            cur_pose = mapper.from_map(x, y, th)
                            trans_prob = odom_motion_model(cur_pose, prev_pose, u)
                            loc.bel_bar[x, y, th] += trans_prob*loc.bel[x_prev, y_prev, th_prev]
    # normalize
    loc.bel_bar = loc.bel_bar/np.sum(loc.bel_bar)
</code></pre>

<pre><code class="language-python">def sensor_model(obs):
    """ This is the equivalent of p(z|x).


    Args:
        obs ([ndarray]): A 1D array consisting of the true observations for a specific robot pose in the map 

    Returns:
        [ndarray]: Returns a 1D array of size 18 (=loc.OBS_PER_CELL) with the likelihoods of each individual sensor measurement
    """

    prob_array = np.zeros(18)
    for i in range(0,mapper.OBS_PER_CELL):
        prob_array[i] = loc.gaussian(lc.obs_range_data[i], obs[i], loc.sensor_sigma)
    
    return prob_array
</code></pre>

<pre><code class="language-python">def update_step():
    """ Update step of the Bayes Filter.
    Update the probabilities in loc.bel based on loc.bel_bar and the sensor model.
    """

    obs = np.empty((mapper.MAX_CELLS_X, mapper.MAX_CELLS_Y, mapper.MAX_CELLS_A, mapper.OBS_PER_CELL))
    obs[:,:,:] = loc.obs_range_data.flatten()
    p = np.prod(loc.gaussian(mapper.obs_views, obs, loc.sensor_sigma), axis = 3)
    prods = np.multiply(p, loc.bel_bar)
    loc.bel = prods/np.sum(prods)

    loc.bel = loc.bel/np.sum(loc.bel)
</code></pre>

<p style="text-align:center;"><iframe width="800" height="600" src="https://www.youtube.com/embed/k_Uz_awCz0Q" allowfullscreen></iframe></p>

<p style="text-align:center;"><img src="..\assets\images\10\Trajectory.png" width="800"/></p>
<p style="text-align:center;"><img src="..\assets\images\10\Data.png" width="800"/></p>

## Conclusion

The bayes filter probably works best in more open or structured regions where the robot likely has strong sensor observations or landmarks to match to. Belief deteriorates in areas with fewer unique features, or in the end when there is an accumulation of heading drift. Overall the belief of the robots position is pretty accurate. The odometry, however, is completely off.




