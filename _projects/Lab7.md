---
layout: project
title: Lab 7
description: Lab 7 Writeup
permalink: /lab7/
---

## Dynamics of the Car

To build the state space model of the system, I first found the drag and momentum terms for the A and B matrices by giving the car a step response. I drove the car towards a wall at the max speed (100 PWM) and logged the ToF sensor distance measurements. To prevent the car from crashing into the wall, I applied active breaking near the end for about half a meter.

<pre><code class="language-cpp">case GET_DM: {
  Serial.println("GET_DM");
  ToFindex=0;
  startTime = millis();
  distanceSensor1.setDistanceModeLong();
  distanceSensor1.startRanging();
  getDM = true;
  driveStraight(maxSpeed);
  break;
}
</code></pre>

In the main loop:

<pre><code class="language-cpp">if(getDM){
  int dist;
  if (distanceSensor1.checkForDataReady()) {
    dist = distanceSensor1.getDistance();
    //Serial.println("forward max speed");
    if(ToFindex < TOF_SAMPLES){
      toftime_arr[ToFindex]=millis();
      ToF_distance[ToFindex]=dist;
    }
    if(dist<300){
      //Serial.println("stop");
      stopDriving();
      
    } else if(dist<800){
      //Serial.println("active break");
      driveStraight(-maxSpeed);
    }

    if(dist<10){
      getDM=false;
      stopDriving();               

    }
    setDistanceMode(dist);
    distanceSensor1.clearInterrupt();
    distanceSensor1.stopRanging();
    distanceSensor1.startRanging();
    ToFindex++;
  }
  
}
</code></pre>

Below shows the car's distance from the wall as it drives towards the wall. There's a slight discontinuity, which causes the velocity graph to spike, probably caused by an noise in distance measurement from the ToF sensor.
<p style="text-align:center;"><img src="..\assets\images\7\OLSR_Distance.png" width="850"/></p>

Below is the plot of the velocity while driving towards the wall. The "steady state" velocity would be the max speed - I decided to ignore the spike in the velocity. The second plot zooms in on the max speed, which is 3.6 m/s.
<p style="text-align:center;"><img src="..\assets\images\7\OLSR_Velocity.png" width="850"/></p>
<p style="text-align:center;"><img src="..\assets\images\7\OLSR_VelocityZoom.png" width="850"/></p>

The system's dynamics are derived as shown below:
<p style="text-align:center;"><img src="..\assets\images\7\Equations1.png" width="200"/></p>

At constant speed, we can find the drag d:
<p style="text-align:center;"><img src="..\assets\images\7\Equations2.png" width="200"/></p>

Since we drove the car at a constant max speed towards the wall, we can use this speed to calculate d (u=1):
<p style="text-align:center;"><img src="..\assets\images\7\d_calc.png" width="200"/></p>

We can also derive m from the last equation using this:
<p style="text-align:center;"><img src="..\assets\images\7\firstordersys.png" width="200"/></p>


...and determine it using the rise time:
<p style="text-align:center;"><img src="..\assets\images\7\v_calc.png" width="200"/></p>

In the velocity plot, we can see that the car's speed is 2.6 m/s (70% rise time) at time t_0.7 = 1973.5 ms = 1.97 s.
<p style="text-align:center;"><img src="..\assets\images\7\m_calc.png" width="400"/></p>

We can organize the dynamics of the system into the following state space form:
<p style="text-align:center;"><img src="..\assets\images\7\genStateSpace.png" width="200"/></p>
<p style="text-align:center;"><img src="..\assets\images\7\state_space.png" width="350"/></p>

Thus, the state space matrices are:
<p style="text-align:center;"><img src="..\assets\images\7\ABC_matrices.png" width="500"/></p>

## Kalman Filter

Below is the Kalman Filter algorithm:
<p style="text-align:center;"><img src="..\assets\images\7\kf_algorithm.png" width="700"/></p>

In python, I wrote a function that implements the Kalman Filter. The function takes in the previous state (mu) and covariance (sigma), as well as the control input (u) and measurement (y).  

<pre><code class="language-python">def kf(mu, sigma, u, y, update = True):
    mu_p = Ad.dot(mu) + Bd.dot(u)
    sigma_p = Ad.dot(sigma.dot(Ad.transpose())) + Sigma_u

    if not update:
        return mu_p, sigma_p

    sigma_m = C.dot(sigma_p.dot(C.transpose())) + Sigma_z
    kkf_gain = sigma_p.dot(C.transpose().dot(np.linalg.inv(sigma_m)))

    y_m = y - C.dot(mu_p)
    mu = mu_p + kkf_gain.dot(y_m)
    sigma = (np.eye(2) - kkf_gain.dot(C)).dot(sigma_p)

    return mu, sigma
</code></pre>

Then I defined the variables and state space matrices. Ad and Bd are A and B discretized.

<pre><code class="language-python">d = 0.278
m = 0.455
A = np.array([[0,1],[0,-d/m]])
B = np.array([[0],[1/m]])
C = np.array([[1,0]])

n = 2 # dimension of the state space
dt = 0.022 # seconds between readings
Ad = np.eye(n) + dt * A
Bd = dt * B

z = tof_distance
u = interp_pwm

# Sensor noise
s3 = 20
Sigma_z = np.array([[s3**2]]) 

# Process noise
s1 = 80
s2 = 80
Sigma_u = np.array([[s1**2, 0], [0, s2**2]])
</code></pre>

Given the ranging error in the ToF sensor specs below, I set the noise to 20.

<p style="text-align:center;"><img src="..\assets\images\7\tof_performance_long.png" width="700"/></p>
<p style="text-align:center;"><img src="..\assets\images\7\tof_performance_short.png" width="700"/></p>

The process noise is somewhat dependent on the sampling rate. My sampling time is roughly 0.02 s, so the process noise is calculated as follows:
<p style="text-align:center;"><img src="..\assets\images\7\process_noise_calcs.png" width="400"/></p>

I set the noise to 80 because with the friction and slipping, there was probably a bit more process noise than calculated.

Then I ran the Kalman Filter, making sure to set the initial conditions, which are just an initial guess of the state and covariance of the system. The initial state consists of the first measurement for distance, and 0 velocity because assume it is starting from rest.

<pre><code class="language-python">kf_vals = []

# initial conditions
x = np.array([[z[0]],[0]])
s = np.array([[50**2, 0], [0, 50**2]])

for i in range(len(z)):
    x,s = kf(x,s,u[i]/100,z[i],True)
    kf_vals.append(x[0])
</code></pre>

## Results

Below plots the estimate from the Kalman Filter in comparison to the ToF sensor measurements of the distance from the wall. The sensor is not very noisy, and the Kalman Filter does a good job estimating the actual trajectory of the car.

<p style="text-align:center;"><img src="..\assets\images\7\Est_Noise20.png" width="800"/></p>

I also tried increasing the sensor noise to 100, which smoothed out the estimate of the car's distance from the wall.

<p style="text-align:center;"><img src="..\assets\images\7\Est_Noise100.png" width="800"/></p>


