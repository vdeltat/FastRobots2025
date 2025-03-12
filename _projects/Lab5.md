---
layout: project
title: Lab 5
description: Lab 5 Writeup
permalink: /lab5/
---

## Prelab

I created two commands: "START_PID" and "STOP_PID" which I could send over bluetooth to command the robot to start and stop the PID loop. These commands would turn the flag "run_pid" true or false. The PID loop sits in the main loop and runs if run_pid=True.

<pre><code class="language-cpp">case START_PID: {
  //reset variables/indeces = 0
  ...
  distanceSensor1.setDistanceModeLong();
  distanceSensor1.startRanging();
  run_pid = true;
  break;
}

case STOP_PID: {
  Serial.println("STOP_PID");
  run_pid = false;
  stopDriving();
  break;
}
</code></pre>

I also created a command "CHANGE_GAIN" to be able to tune the PID gains on the fly over bluetooth rather than uploading code to the board every time I wanted to change a gain value. This included the PID gains as well as alpha for the derivative term low-pass filter. This made the PID tuning process a lot more efficient.

<pre><code class="language-cpp">case CHANGE_GAIN: {
  Serial.println("CHANGE_GAIN");
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

  Kp = new_Kp;
  Ki = new_Ki;
  Kd = new_Kd;
  d_alpha = new_alpha;

  break;
}
</code></pre>

I also didn't want the PID loop to automatically send the data every time it finished while I repetively tuned the gains, so I created separate commands to do that as well: 

<pre><code class="language-cpp">case SEND_PID_DATA: {
  sendpidData();
  break;
}

case SEND_TOF_DATA: {
  sendToFData();
  break;
}

void sendpidData() {
  for (int j = 0; j < PIDindex; j++) { 
    tx_estring_value.clear();
    float pid_time = (float) pidtime_arr[j]/1000 - (float) startTime; //convert to ms
    tx_estring_value.append(pid_time);
    tx_estring_value.append(" | ");        
    tx_estring_value.append(distance_pid[j]);
    tx_estring_value.append(" | "); 
    tx_estring_value.append(pwm_arr[j]);
    tx_characteristic_string.writeValue(tx_estring_value.c_str());
    delay(5);
  }
}

void sendToFData() {
  for (int i = 0; i < ToFindex; i++) { 
    tx_estring_value.clear();
    char time_str[20];
    sprintf(time_str, "%lu", toftime_arr[i]-startTime);
    tx_estring_value.append(time_str);  
    tx_estring_value.append(" | ");        
    tx_estring_value.append(ToF_distance[i]);
    tx_characteristic_string.writeValue(tx_estring_value.c_str());
    delay(5);
  }
}
</code></pre>

## PID Control

One thing to note is that I set the maximum speed at 100 PWM so the robot would never crash into the wall at too high speed. For practical purposes I set constraints on the speed. If the PID loop wanted to give a control input greater than the maximum PWM, it would set it at the max PWM. If the control input was lower than a certain threshold I set (which I settled finally on 10 PWM), it would set the control input to zero, since a low enough control input was effectively zero and wouldn't really move the robot. However, as we found in lab 4, the robot would not be able move at low PWM values, so I set a minimum speed at 40 PWM. If the resulting control input from the PID controllers was above the zero threshold, but less than the minimum speed, it would set the control input to the minimum speed so that the robot could actually move. Since the integral term can accumulate pretty quickly (integral wind-up) and lead to instability, I also constrained its max constribution to the control input to be 300. I put a low-pass filter on the the derivative term to reduce noise, and tuned d_alpha.

<pre><code class="language-cpp">float lin_pid(int dist, float targetDist) {
  //Serial.println("lin_pid");
  float error;
  float pwm;
  unsigned long current_time = micros();
  float dt = (current_time - prev_time) / 1e6; // Convert microsec to sec
  
  error = dist-targetDist;
  float deriv = (error - prev_error)/dt;
  float filter_deriv = d_alpha * deriv + (1 - d_alpha) * prev_deriv;
  prev_deriv = filter_deriv;

  sumerror = constrain(sumerror + error * dt, -300, 300);

  pwm = Kp*error + Ki*sumerror + Kd*filter_deriv;

  // Speed constraints
  if(abs(pwm) > maxSpeed){
    pwm=copysignf(maxSpeed, pwm);
  } else if (abs(pwm) < zero_thresh){
    pwm=0;
  } 
  else if(abs(pwm) < minSpeed){
    pwm=copysignf(minSpeed, pwm);
  }
  
  prev_error = error;  // Update previous error for next iteration
  prev_time = current_time;
  return pwm;
}
</code></pre>

## Linear Extrapolation

The time of flight sensor had an average timestep of 22.42 ms and the PID loop had an average timestep of 7.60 ms, when I tested the car 3m from the wall. Since the ToF sensor data takes longer to receive than one iteration of the PID loop, the extrapolation function below uses the two previous ToF sensor data to linearly extrapolate an estimated distance at the current time in the PID loop if the ToF sensor data is not available.

<pre><code class="language-cpp">float extrapolateDistance(int d1, int d2, unsigned long t1, unsigned long t2) {
  unsigned long t = millis();  // current time in ms
  
  if (t1 == t2) return d2; //avoid /0

  float v = (float)(d2 - d1) / (float)(t2 - t1); //mm/ms
  float d = d2 + v*(t - t2);
  return d;
}
</code></pre>

## Set Distance Mode

I had first started testing at 2m away from the wall, so I didn't notice that using short mode for the ToF sensor would be a problem until I started testing at 3m away. The sensor would not give a reading when it was that far away, so I was forced to use long mode. The accruacy reduced when I was using long mode and the car would often overshoot, so I created a function that would switch to short mode if it detected that it was closer than 1.3m from the wall, and long mode if it was greater. This function runs every time the ToF sensor gets a measurement in the PID loop. Since it takes a bit of time to switch modes, I used a flag to remember what mode its set to so it doesn't have to set the mode every time it gets a sensor measurement. Although short mode maximum detection distance is technically 1.3m, it would work a little beyond that (like when I was testing at 2m), so this method worked. Switching between the modes for appropriate distances helped improve the accuracy of the measurements at short distances while also being able to detect longer distances.

<pre><code class="language-cpp">void setDistanceMode(int distance){

    if (distance > 0 && distance <= 1300 && distanceMode) {
        distanceSensor1.setDistanceModeShort();
        distanceMode = 0;        //Serial.println("Mode: Short");
        distanceSensor1.setTimingBudgetInMs(15);
    }
    // If distance is invalid (0) or out of range in Short mode, switch back to Long mode
    else if ((distance == 0 || distance > 1300) && distanceMode == 0) {
        distanceSensor1.setDistanceModeLong();
        distanceMode = 1;
        //Serial.println("Mode: Long");
        distanceSensor1.setTimingBudgetInMs(33);
    }
}
</code></pre>

## PID Loop

Synthesizing the parts above, the below code shows the PID loop within the main loop. It first collects the ToF sensor data if available, or else extrapolates a distance estimate based on previous sensor data. Then the PID controller calculates the control input which is then used to control the motors.

<pre><code class="language-cpp">if (run_pid){
  int dist;
  float pwm;

  if (distanceSensor1.checkForDataReady()) {
    dist = distanceSensor1.getDistance();

    setDistanceMode(dist);

    if(ToFindex < TOF_SAMPLES){
      toftime_arr[ToFindex]=millis();
      ToF_distance[ToFindex]=dist;
    }
    distanceSensor1.clearInterrupt();
    distanceSensor1.stopRanging();
    distanceSensor1.startRanging();
    ToFindex++;
  } else { //extrapolate
    if(ToFindex>1){
      dist = extrapolateDistance(ToF_distance[ToFindex-1],ToF_distance[ToFindex-2],toftime_arr[ToFindex-1],toftime_arr[ToFindex-2]);
    } else {
      dist = 1500;
    }                
  }

  pwm = lin_pid(dist, TARGET_DIST);

  // collect data
  if(PIDindex < PID_SAMPLES){
    pidtime_arr[PIDindex] = micros();
    distance_pid[PIDindex] = dist;
    pwm_arr[PIDindex] = pwm;
  }
  PIDindex++;

  if(pwm > 0){
    driveStraight(1,pwm);
  } else if(pwm < 0){
    driveStraight(0,abs(pwm));
  } else {
    stopDriving();
  }                    
}
</code></pre>

## PID Gain Discussion
I started with tuning Kp (proportional gain), setting the other two gains to 0. I had set the default value Kp = 1, which was way too high so I lowered it. Whenever I noticed a overshoot, I would increase Kd (derivative gain). Increasing Kd would sometimes cause the robot to shudder as it drove if it was too high, so I decreased the alpha for the low-pass filter to weight the previous values more and that helped smooth the control. Whenever the robot slowed and stopped before reaching 1 foot from the wall, I increased Ki (integral gain). 

<p style="text-align:center;"><iframe width="800" height="600" src="https://www.youtube.com/embed/J44NtmZatdw" allowfullscreen></iframe></p>

<p style="text-align:center;"><iframe width="800" height="600" src="https://www.youtube.com/embed/5OpC3tmyL0s" allowfullscreen></iframe></p>

## Final Results

My final gain values are: Kp = 0.04, Ki = 0.047, Kd = 0.03, as well as alpha = 0.07. Below shows videos and graphs of the final result of the car driving from 2m, 3m, and 4m away from the wall, stopping at approximately one foot from the wall. The PWM vs. Time graphs shows the control input the motors are given. The ToF Sensor Distance graphs show the sensor's distance measurements over time. The PID Loop Distance vs. Time graphs show the extrapolated distances that the PID controller uses to calculate the control input. 

### Starting 2 Meters Away from Wall

<p style="text-align:center;"><iframe width="800" height="600" src="https://www.youtube.com/embed/vxkT8VmauhU" allowfullscreen></iframe></p>

<p style="text-align:center;"><img src="..\assets\images\5\2m_PWM.png" width="950"/></p>

<p style="text-align:center;"><img src="..\assets\images\5\2m_PID_Distance.png" width="950"/></p>

<p style="text-align:center;"><img src="..\assets\images\5\2m_TOF.png" width="950"/></p>

### Starting 3 Meters Away from Wall

<p style="text-align:center;"><iframe width="800" height="600" src="https://www.youtube.com/embed/ZYdS-cv6Er0" allowfullscreen></iframe></p>

<p style="text-align:center;"><img src="..\assets\images\5\3m_PWM.png" width="950"/></p>

<p style="text-align:center;"><img src="..\assets\images\5\3m_PID_Distance.png" width="950"/></p>

<p style="text-align:center;"><img src="..\assets\images\5\3m_TOF.png" width="950"/></p>

### Starting 4 Meters Away from Wall

<p style="text-align:center;"><iframe width="800" height="600" src="https://www.youtube.com/embed/31SfJLWSqmg" allowfullscreen></iframe></p>

<p style="text-align:center;"><img src="..\assets\images\5\4m_PWM.png" width="950"/></p>

<p style="text-align:center;"><img src="..\assets\images\5\4m_PID_Distance.png" width="950"/></p>

<p style="text-align:center;"><img src="..\assets\images\5\4m_TOF.png" width="950"/></p>