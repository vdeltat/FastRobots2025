---
layout: project
title: Lab 6
description: Lab 6 Writeup
permalink: /lab6/
---

## Prelab

I created two commands: "START_ORI_PID" and "STOP_ORI_PID" which I could send over bluetooth to command the robot to start and stop the orientation PID loop. These commands would turn the flag "run_ori_pid" true or false. The orientation PID loop sits in the main loop and runs if run_ori_pid = true.

<pre><code class="language-cpp">case START_PID: {
  //reset variables/indeces = 0
  ...
  run_ori_pid = true;
  break;
}

case STOP_PID: {
  run_ori_pid = false;
  stopDriving();
  break;
}
</code></pre>

I also created a command "CHANGE_ORI_GAIN" to be able to tune the PID gains on the fly over bluetooth rather than uploading code to the board every time I wanted to change a gain value. This included the PID gains as well as alpha for the derivative term low-pass filter. This made the PID tuning process a lot more efficient.

The command "SET_ORIENTATION" sets the target setpoint angle to the angle given as an input.

<pre><code class="language-cpp">case CHANGE_GAIN: {
  float new_Kp; float new_Ki; float new_Kd; float new_alpha;
          
  success = robot_cmd.get_next_value(new_Kp);
  if(!success)
    return;

  success = robot_cmd.get_next_value(new_Ki);
  if(!success)
    return;

  success = robot_cmd.get_next_value(new_Kd);
  if(!success)
    return;
  
  success = robot_cmd.get_next_value(new_alpha);
  if(!success)
    return;

  Kp_ori = new_Kp;
  Ki_ori = new_Ki;
  Kd_ori = new_Kd;
  alpha_ori = new_alpha;

  break;
}
</code></pre>

I also didn't want the PID loop to automatically send the data every time it finished while I repetively tuned the gains, so I created separate commands to do that as well: 

<pre><code class="language-cpp">case SEND_PID_DATA_ORI: {
  sendpidData_Ori();
  break;
}

void sendpidData_Ori() {
  for (int j = 0; j < PIDindex_ori; j++) { 
    tx_estring_value.clear();
    //char time_str[20];
    //sprintf(time_str, "%lu", pidtime_arr[j]);
    float pid_time = (float) ori_pidtime[j]/1000 - (float) startTime; //convert to ms
    tx_estring_value.append(pid_time);
    tx_estring_value.append(" | ");        
    tx_estring_value.append(ori_pidorient[j]);
    tx_estring_value.append(" | "); 
    tx_estring_value.append(ori_pwm_arr[j]);
    tx_characteristic_string.writeValue(tx_estring_value.c_str());
    delay(5);
  }
}
</code></pre>

## PID Control for Orientation

For practical purposes I set constraints on the speed. If the PID loop wanted to give a control input greater than the maximum PWM of 250, it would set it at the max PWM. As we found in lab 4, the robot would not be able move at low PWM values, so I set a minimum turning speed at 190 PWM. If the resulting control input from the PID controllers was above the zero threshold, but less than the minimum speed, it would set the control input to the minimum speed so that the robot could actually move. If the control input was lower than a the zero threshold (100 PWM), it would set the control input to zero, since a low enough control input was effectively zero and wouldn't really turn the robot. I put a low-pass filter on the the derivative term to reduce noise, and tuned the alpha value.

<pre><code class="language-cpp">float ori_pid(float targetOrient) {
  float pwm;
  unsigned long current_time = micros();
  float dt = (current_time - prev_time) / 1e6; // Convert microsec to sec
  
  float error = yaw-targetOrient;
  
  // Set target reciprocal at 180 degrees offset
  float pid_ori_recip = targetOrient - (targetOrient < 0 ? -1 : 1) * 180;

  if (abs(error) > 180.0) {
      // Prevent sudden jumps in angle when crossing the +/- 180 boundary
      error += (pid_ori_recip - yaw) / abs(pid_ori_recip - yaw) * 360.0;
  }

  float deriv = (error - prev_error_ori)/dt;
  float filter_deriv = alpha_ori * deriv + (1 - alpha_ori) * prev_deriv_ori;
  prev_deriv_ori = filter_deriv;

  pwm = Kp_ori*error + Ki_ori*sumerror_ori + Kd_ori*filter_deriv;

  // Speed constraints
  if(abs(pwm) > maxTurnSpeed){
    pwm=copysignf(maxTurnSpeed, pwm);
  } 
  else if (abs(pwm) < zero_turn_thresh){
    pwm=0;
  }
  else if(abs(pwm) < minTurnSpeed){
    pwm=copysignf(minTurnSpeed, pwm);
  }
  
  prev_error_ori = error;  // Update previous error for next iteration
  prev_time = current_time;
  return pwm;
}
</code></pre>

## PID Loop

Synthesizing the parts above, the below code shows the PID loop within the main loop. It first collects the ToF sensor data if available, or else extrapolates a distance estimate based on previous sensor data. Then the PID controller calculates the control input which is then used to control the motors.

<pre><code class="language-cpp">if(run_ori_pid){
  float ori_pwm;

  if (myICM.dataReady()) {
    getOrientation();
  }

  ori_pwm = ori_pid(TARGET_ORIENT);

  // collect data
  if(PIDindex_ori < PID_SAMPLES){
    ori_pidtime[PIDindex_ori] = micros(); // store micros
    ori_pidorient[PIDindex_ori] = yaw;
    ori_pwm_arr[PIDindex_ori] = ori_pwm;
  }
  PIDindex_ori++;

  drivePointTurn(ori_pwm);              
}
</code></pre>

## PID Gain Discussion
I started with tuning Kp (proportional gain), setting the other two gains to 0. Sometimes it wouldn't reach the setpoint angle, so I increased the Ki (integral gain term). When it was too high, it would cause the robot to overshoot and oscillate a lot or go unstable, so I would increase the Kd (derivative gain).

## Final Results

My final gain values are: Kp = 2, Ki = 1.5, Kd = 1, as well as alpha = 0.1. 

### Disturbance Correction

Below shows the response from the car when disturbed from its target angle. The orientation is the angle given in degrees.

<p style="text-align:center;"><iframe width="420" height="600" src="https://www.youtube.com/embed/WuonS_ccF9w" allowfullscreen></iframe></p>
<p style="text-align:center;"><img src="..\assets\images\6\Disturbance-PWM.png" width="900"/></p>
<p style="text-align:center;"><img src="..\assets\images\6\Disturbance-orientation.png" width="900"/></p>

### Setpoint Changes

Below shows the car rotating to each setpoint angle I give: 45 deg, 90 deg, 135 deg, 180 deg, and 225 deg.

<p style="text-align:center;"><iframe width="420" height="600" src="https://www.youtube.com/embed/vco97wLjZz8" allowfullscreen></iframe></p>
<p style="text-align:center;"><img src="..\assets\images\6\Setpoint-PWM.png" width="900"/></p>
<p style="text-align:center;"><img src="..\assets\images\6\Setpoint-orientation.png" width="900"/></p>

