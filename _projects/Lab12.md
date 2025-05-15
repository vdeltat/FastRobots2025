---
layout: project
title: Lab 12
description: Lab 12 Writeup
permalink: /lab12/
---

## Inverted Pendulum

I tried to balance the car on its two rear wheels for this lab. I used the same code from lab 6 for yaw orientation, but with a few changes. Instead of turning which requires the wheels to turn in opposite directions, I commanded the car to drive straight. The PID controller would give an appropriate control signal that would balance the car - 90 degrees roll for the target orientation in this case.

<pre><code class="language-cpp">case START_BALANCE: {
    PIDindex_ori=0;
    prev_error_ori=0;
    sumerror_ori=0;
    prev_deriv_ori=0;

    startTime = millis();
    prev_time = micros();
    balance=true;
    break;
}
</code></pre>

<pre><code class="language-cpp">if(balance){
    
    //Serial.println("Run ori pid");
    float roll_pwm;

    if (myICM.dataReady()) {
    getOrientation();
    }
    
    roll_pwm = roll_pid(TARGET_ORIENT);

    // collect data
    if(PIDindex_ori < PID_SAMPLES){
    ori_pidtime[PIDindex_ori] = micros(); // store micros
    ori_pidorient[PIDindex_ori] = roll;
    ori_pwm_arr[PIDindex_ori] = roll_pwm;
    }
    PIDindex_ori++;
    
    driveStraight(roll_pwm);              
}
</code></pre>

As in lab 6, I used the DMP roll measurement. Instead of calculating the derivative term, I used the gyroscope measurement (angular velocity). I also discovered that I had commented out the integral term so that might have been why lab 11 was not working well.



<pre><code class="language-cpp">float roll_pid(float targetOrient) {
    float pwm;
    unsigned long current_time = micros();
    float dt = (current_time - prev_time) / 1e6; // Convert microsec to sec
    
    float error = roll-targetOrient;
    
    // Set target reciprocal at 180 degrees offset
    float pid_ori_recip = targetOrient - (targetOrient < 0 ? -1 : 1) * 180;

    if (abs(error) > 180.0) {
        // Prevent sudden jumps in angle when crossing the +/- 180 boundary
        error += (pid_ori_recip - roll) / abs(pid_ori_recip - roll) * 360.0;
    }

    //error = fmod((error + 180), 360) - 180;

    float deriv;

    if (myICM.dataReady()) {
        myICM.getAGMT();  // Reads all sensor values
        deriv = (float) myICM.gyrX();  // Gyro roll rate in deg/sec
    } else {
        deriv = (error - prev_error_ori)/dt;
    }

    float filter_deriv = alpha_ori * deriv + (1 - alpha_ori) * prev_deriv_ori;
    prev_deriv_ori = filter_deriv;

    sumerror_ori = constrain(sumerror_ori + error * dt, -300, 300);

    pwm = Kp_ori*error + Ki_ori*sumerror_ori + Kd_ori*filter_deriv;

    // Speed constraints
    if(abs(pwm) > maxBalanceSpeed){
        pwm=copysignf(maxBalanceSpeed, pwm);
    } 
    
    else if (abs(pwm) < zero_balance_thresh){
        pwm=0;
    }
    else if(abs(pwm) < minBalanceSpeed){
        pwm=copysignf(minBalanceSpeed, pwm);
    }
    
    prev_error_ori = error;  // Update previous error for next iteration
    prev_time = current_time;
    
    return pwm;

}
</code></pre>

## Results

It doesn't work great. I would probably add a Kalman Filter if I had more time because it was very noisy and jittery - the quality of the measurements were probably not great so the controller wasn't able to balance it very well. It was also somewhat dependent on the battery, and the motor controllers would get very hot after a few times of testing. Below are some short instances of it working.

<p style="text-align:center;"><iframe width="420" height="600" src="https://www.youtube.com/embed/B9R9EAJGrWs" allowfullscreen></iframe></p>
<p style="text-align:center;"><iframe width="420" height="600" src="https://www.youtube.com/embed/GvSrARsnbzA" allowfullscreen></iframe></p>



