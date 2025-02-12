---
layout: project
title: Lab 2
description: Lab 2 Writeup
permalink: /lab2/
---

## Setup the IMU

The AD0_VAL defines the value of pin AD0 that determines the LSB bit of the 7-bit slave address of the ICM-20948. In other words, it controls the X bit in b110100X.

<p style="text-align:center;"><img src="..\assets\images\2\p1_1.png" width="300"/></p>

<p style="text-align:center;"><img src="..\assets\images\2\p1_2.png" width="950"/></p>

The units for acceleration is mg or “milli-g’s” or 1 mg = 0.00981 m/s², since 1000 mg = 1 g. When I keep the board at rest on the table, the acceleration in the z direction is 1 g, which makes sense because I am pointing the z-direction of the board in the direction of gravity. The accelerations in the other two directions are basically 0, with some deviation due to noise, vibrations, or have a negligent component of gravity in that direction. If I move the board in the x or y direction, the acceleration in the respective direction is shown to increase, as expected.

<p style="text-align:center;"><img src="..\assets\images\2\p1_3.png" width="950"/></p>

When I tilt the board about the x-axis, the y-direction gains a component of gravity and increases its acceleration in y while decreasing the acceleration in z, and vice versa about the y-axis.

The gyroscope data in deg/s gives the angular velocity of the board about each axis respectively. So for example, as I rotate the board about the x-axis, the first value in the reading gives a value for the angular velocity I’m rotating the board at.

I added code to have to board blink the LED three times slowly on start-up.

<p style="text-align:center;"><iframe width="420" height="600" src="https://www.youtube.com/embed/VCYls_c7HTA" allowfullscreen></iframe></p>

## Accelerometer


| True Angle | Measured Pitch | Measured Roll |
| ---------- | -------------- | ------------- |
|    -90     |     -89.47     |    -90.68     |
|     0      |      0.22      |     -0.25     |
|     90     |      87.57     |     88.68     |

Using this data, I performed a two-point calibration as described here: https://learn.adafruit.com/calibrating-sensors/two-point-calibration

Pitch:

Reference Range = 90 - (-90) = 180

Raw Range = 87.57 - (-89.47) = 177.04

Corrected Value =  1.017 * (Raw Value + 89.47) - 90

Roll:

Reference Range = 90 - (-90) = 180

Raw Range = 88.68 - (-90.68) = 179.36

Corrected Value = 1.00357 * (Raw Value + 90.68) - 90

The two-point calibration method provides a corrected value based on the given raw value. Usin this method, these equations above provide the calibrated sensor values. The accelerometer is very accurate as shown through the calibration factor being very close to 1. Additionally, the accelerometer measurements are very close to the reference values, being off by at most 2.5 degrees.

### Noise

<p style="text-align:center;"><img src="..\assets\images\2\p2_fftPitch.png" width="700"/></p>
<p style="text-align:center;"><img src="..\assets\images\2\p2_fftRoll.png" width="700"/></p>

### Low Pass Filter
<p style="text-align:center;"><img src="..\assets\images\2\p2_PitchLPF.png" width="550"/></p>
<p style="text-align:center;"><img src="..\assets\images\2\p2_RollLPF.png" width="550"/></p>
