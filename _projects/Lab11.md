---
layout: project
title: Lab 11
description: Lab 11 Writeup
permalink: /lab11/
---

## Simulation

I tested the simulation code and below shows the results.

<p style="text-align:center;"><img src="..\assets\images\11\sim_plot.png" width="800"/></p>

## Real

I planned to use the code below to gather distance data automatically instead of incrementing and collecting data manually for each angle like I did in lab 9, but got stuck on getting it to actually work well. I ran out of time for this lab so I didn't get to complete the rest of it.

<pre><code class="language-cpp">case MAP: {
    distanceSensor1.setDistanceModeShort();
    distanceSensor1.startRanging();
    mapping = true; // mapping flag
    angIndex=0;
    ori_reached = false;
    TARGET_ORIENT = target_angles[angIndex];
}
</code></pre>

In the main loop:
<pre><code class="language-cpp">if(mapping){
    float ori_pwm;
    int dist_samples=10;
    float dist_sum=0;
    float dist;

    if (myICM.dataReady()) {
    getOrientation();
    }

    ori_pwm = ori_pid(TARGET_ORIENT);
    //Serial.println("ori_pwm");     

    if (ori_reached) {
    Serial.println("ori_reached");
    stopDriving();
    delay(1000);
    for (int i = 0; i < dist_samples; i++) {
        while (!distanceSensor1.checkForDataReady()) {
        delay(1);
        }
        dist_sum += distanceSensor1.getDistance();
    }
    dist = dist_sum/dist_samples;

    // collect data
    if(angIndex < TURNS){
        map_angles[angIndex] = yaw;
        map_dists[angIndex] = dist;
        //next turn
        angIndex++;
        TARGET_ORIENT = target_angles[angIndex];
        ori_reached = false;
        Serial.println("collected data");
    } else {
        mapping = false;
        Serial.println("done mapping");
    }

    distanceSensor1.clearInterrupt();
    distanceSensor1.stopRanging();
    distanceSensor1.startRanging();
    } else {
    drivePointTurn(ori_pwm); 

    }  

}
</code></pre>




