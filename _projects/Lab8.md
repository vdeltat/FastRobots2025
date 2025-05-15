---
layout: project
title: Lab 8
description: Lab 8 Writeup
permalink: /lab8/
---

## Objective

The objective of this lab was to get a car to drive a max speed towards a wall, flip about 1 foot from the wall, then drive back to where it started.

## Procedure

I hardcoded in the following steps:

1) Drive forwards at max speed (PWM of 255) for 600 ms
2) Drive backwards at max speed (PWM of -255) for 1200 ms
3) Stop driving

Code:
<pre><code class="language-cpp">case FLIP: {
  driveStraight(255);
  delay(600);
  driveStraight(-255);
  delay(1200);
  stopDriving();
  break;
}
</code></pre>

Driving forward and then immediately drive backwards while adding a weight to the front of the car made the front of the car the pivot point of the flip.

I planned on using the ToF sensor to tell the car to drive backwards when it was 1 foot from the wall, but it kept crashing into the wall. Initially I thought that the momentum of the car was too high and it would coast after it measures 1 foot, but I realize now that it could be due to a different reason. I had forgotten to uncomment my extrapolation of the distance data when I had commented out for the previous lab (Kalman Filter). I hadn't gotten the Kalman Filter to work well on the car itself, so it was useless for this lab. However the extrapolation might have helped to increase the frequency in distance measurements and may have prevented crashing into the wall. 

I completed this lab the day before the ECE Robotics Showcase, so there wasn't much time to debug using the distance measurements to complete the lab, so I just coded the flip with timed delays.

## Results
This video was taken at the showcase.
<p style="text-align:center;"><iframe width="420" height="600" src="https://www.youtube.com/embed/c6ALx52YUrE" allowfullscreen></iframe></p>






