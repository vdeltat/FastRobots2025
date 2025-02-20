---
layout: project
title: Lab 2
description: Lab 2 Writeup
permalink: /lab2/
---

## Setup the IMU

The AD0_VAL defines the value of pin AD0 that determines the LSB bit of the 7-bit slave address of the ICM-20948. In other words, it controls the X bit in b110100X.

<p style="text-align:center;"><img src="..\assets\images\2\p1_1.png" width="200"/></p>

<p style="text-align:center;"><img src="..\assets\images\2\p1_2.png" width="950"/></p>

The units for acceleration is mg or “milli-g’s” or 1 mg = 0.00981 m/s², since 1000 mg = 1 g. When I keep the board at rest on the table, the acceleration in the z direction is 1 g, which makes sense because I am pointing the z-direction of the board in the direction of gravity. The accelerations in the other two directions are basically 0, with some deviation due to noise, vibrations, or have a negligent component of gravity in that direction. If I move the board in the x or y direction, the acceleration in the respective direction is shown to increase, as expected.

<p style="text-align:center;"><img src="..\assets\images\2\p1_3.png" width="950"/></p>

When I tilt the board about the x-axis, the y-direction gains a component of gravity and increases its acceleration in y while decreasing the acceleration in z, and vice versa about the y-axis.

The gyroscope data in deg/s gives the angular velocity of the board about each axis respectively. So for example, as I rotate the board about the x-axis, the first value in the reading gives a value for the angular velocity I’m rotating the board at.

I added code to have to board blink the LED three times slowly on start-up.

<p style="text-align:center;"><iframe width="420" height="600" src="https://www.youtube.com/embed/VCYls_c7HTA" allowfullscreen></iframe></p>

## Accelerometer

To compute pitch and roll angles using accelerometer measurements, I implemented the following equations:

<pre><code class="language-cpp">pitch_a = atan2(myICM.accX(), myICM.accZ()) * 180 / M_PI; 
roll_a  = atan2(myICM.accY(), myICM.accZ()) * 180 / M_PI;
</code></pre>

| True Angle | Measured Pitch | Measured Roll |
| ---------- | -------------- | ------------- |
|    -90     |     -89.47     |    -90.68     |
|     0      |      0.22      |     -0.25     |
|     90     |      87.57     |     88.68     |

Using this data, I performed a two-point calibration as described here: https://learn.adafruit.com/calibrating-sensors/two-point-calibration

Pitch:

* Reference Range = 90 - (-90) = 180
* Raw Range = 87.57 - (-89.47) = 177.04
* Corrected Value =  1.017 * (Raw Value + 89.47) - 90

Roll:

* Reference Range = 90 - (-90) = 180
* Raw Range = 88.68 - (-90.68) = 179.36
* Corrected Value = 1.00357 * (Raw Value + 90.68) - 90

The two-point calibration method provides a corrected value based on the given raw value. Using this method, these equations above provide the calibrated sensor values. The accelerometer is very accurate as shown through the calibration factor being very close to 1. Additionally, the accelerometer measurements are very close to the reference values, being off by at most 2.5 degrees.

### Noise

<p style="text-align:center;"><img src="..\assets\images\2\p2_fftPitch.png" width="700"/></p>
<p style="text-align:center;"><img src="..\assets\images\2\p2_fftRoll.png" width="700"/></p>

The FFT shows that most of the signal consists of the lower few frequencies. The higher frequencies constitute the noise in the signal, so we want to cut off the the higher frequencies with a low pass filter. Since most of the signal is under 5 Hz, I set that as the cutoff.

### Low Pass Filter

To implement the low pass filter, we need to find the value alpha:

<p style="text-align:center;"><img src="..\assets\images\2\eq1.png" width="150"/></p>
<p style="text-align:center;"><img src="..\assets\images\2\eq2.png" width="150"/></p>

where T = Sampling Period, and f_c = cutoff frequency. The alpha for my case is calculated to be 0.039. This filter is implemented as such:

<pre><code class="language-cpp">pitch_a = atan2(myICM.accX(), myICM.accZ()) * 180 / M_PI; 
roll_a  = atan2(myICM.accY(), myICM.accZ()) * 180 / M_PI; 

const float alpha = 0.039;
pitch_LPF[i] = alpha*pitch_a + (1-alpha)*pitch_LPF[i-1];
pitch_LPF[i-1] = pitch_LPF[i];

roll_LPF[i] = alpha*roll_a + (1-alpha)*roll_LPF[i-1];
roll_LPF[i-1] = roll_LPF[i];
</code></pre>

<p style="text-align:center;"><img src="..\assets\images\2\p2_PitchLPF.png" width="550"/></p>
<p style="text-align:center;"><img src="..\assets\images\2\p2_RollLPF.png" width="550"/></p>

The filtered signal is more smooth, with the higher freqencies filtered out.

## Gyroscope

To compute pitch, roll, and yaw angle data using gyroscope measurements, I implemented the following equations:

<pre><code class="language-cpp">pitch_g = pitch_g + myICM.gyrX()*dt; 
roll_g = roll_g + myICM.gyrY()*dt;
yaw_g = yaw_g + myICM.gyrZ()*dt;
</code></pre>

As shown in the diagram of the ICM 20948 below, the coordinate system labeled "AG" stand for Accelerometer and Gyro, so they share the same axes.
<p style="text-align:center;"><img src="..\assets\images\2\ICM20948.png" width="300"/></p>

Below shows the results of the angle data computed using the gyroscope measurements compared with the angles computes using the accelerometer measurements, as well it filtered.

<p style="text-align:center;"><img src="..\assets\images\2\gyro_pitch.png" width="550"/></p>
<p style="text-align:center;"><img src="..\assets\images\2\gyro_roll.png" width="550"/></p> 
<p style="text-align:center;"><img src="..\assets\images\2\gyro_yaw.png" width="550"/></p>

The gyro doesn't really have any absolute reference, so it relies on changes in angle. That is why it is offset by a pretty consistent value from the angles resulting from the accelerometer. You can also see in the yaw that the gyro can tend to drift. Since the angles computed using the gyro data is dependent on "dt," the accuracy of the of gyroscope angles will increase with sampling frequency (or a lower dt). The gyroscope is also less susceptible to noise.

### Complementary Filter

I implemented a complementary filter to fuse the orientation computed using gyroscope data with the that computed using accelerometer data. I chose alpha_cf = 0.8 because the filtered accelerometer values are more accurate, so I want to trust those more over the gyroscope values.

<pre><code class="language-cpp">pitch_cf[i] = pitch_LPF[i]*(alpha_cf) + pitchg_arr[i]*(1-alpha_cf);
roll_cf[i] = roll_LPF[i]*(alpha_cf) + rollg_arr[i]*(1-alpha_cf);
</code></pre>

<p style="text-align:center;"><img src="..\assets\images\2\comp_pitch.png" width="550"/></p>
<p style="text-align:center;"><img src="..\assets\images\2\comp_roll.png" width="550"/></p> 

The complementary filter helps mitigates the vibrations and spikes in the signal with the accelerometer, as well as the drift in the gyroscope.

## Sample Data

To speed up the code, I removed all Serial.print statements. Instead of waiting for the IMU data to be ready before collecting data, I checked if data is ready in every iteration of the main loop, where it would collect and store it in the arrays whenever it was ready. I also reduced the arrays to the final time, pitch, roll, and yaw float arrays. Floats store decimals unlike integers, but take less space than doubles, although a bit less precise. 

Three float arrays of the same size N take up 3 x 4 bytes x N, where I used N=2000. It took 6395 ms to collect 24000 bytes of data, so approximately 3753 bytes/s. Let's say 3/4 of the board's RAM is available, 288 kB. This would mean the Artemis can store about 75 seconds of IMU data.
 
The two commands below were added to let the user decide when to start and stop recording data.

<pre><code class="language-cpp">case  START_RECORD_DATA: {
    collect = true;
    Serial.println("Start collecting IMU data");
    break;
}
</code></pre>

<pre><code class="language-cpp">case  STOP_RECORD_DATA: {
    collect = false;
    Serial.println("Stop collecting IMU data. Sending data...");
    sendIMUdata();
    Serial.println("Done sending data");
    break;
}
</code></pre>

Then in the main loop:

<pre><code class="language-cpp">while (central.connected()) {
// Send data
write_data();

if (myICM.dataReady() && collect) {
    getIMUdata();
}
// Read data
read_data();
}
</code></pre>

The final code to collect the IMU data:

<pre><code class="language-cpp">void getIMUdata(){
  float pitcha;
  float rolla;

  if (IMUindex >= SAMPLES){
    Serial.println("Array full");
    collect = false;
    return;
  }

  // Collect Data
  myICM.getAGMT();
  if (IMUindex==0){
    start_time = millis();
    time_arr[IMUindex] = 0;
  } else {
    time_arr[IMUindex] = millis()-start_time;
  }

  pitcha = atan2(myICM.accX(), myICM.accZ()) * 180 / M_PI; 
  rolla  = atan2(myICM.accY(), myICM.accZ()) * 180 / M_PI; 

  pitchlpf = alpha*pitcha + (1-alpha)*pitchlpf_prev;
  pitchlpf_prev = pitchlpf;

  rolllpf = alpha*rolla + (1-alpha)*rolllpf_prev;
  rolllpf_prev = rolllpf;
  
  // Gyro
  deltat = (micros()-last_time)/1000000.;
  last_time = micros();

  // Gyro shares same axes as acc
  pitchg = pitchg + myICM.gyrX()*deltat; 
  rollg = rollg + myICM.gyrY()*deltat;
  yawg = yawg + myICM.gyrZ()*deltat;

  // Complementary filter
  pitch_arr[IMUindex] = pitchlpf*(alpha_cf) + pitchg*(1-alpha_cf);
  roll_arr[IMUindex] = rolllpf*(alpha_cf) + rollg*(1-alpha_cf);
  yaw_arr[IMUindex] = yawg;
  IMUindex++;
}
</code></pre>

After the IMU data is collected, "sendIMUdata()" is run:

<pre><code class="language-cpp">void sendIMUdata() {
  for (int j = 0; j < IMUindex; j++) { 
    tx_estring_value.clear();
    char time_str[20];
    sprintf(time_str, "%lu", time_arr[j]);
    tx_estring_value.append(time_str);
    tx_estring_value.append(" | ");
    tx_estring_value.append(pitch_arr[j]);
    tx_estring_value.append(" | ");
    tx_estring_value.append(roll_arr[j]);
    tx_estring_value.append(" | ");            
    tx_estring_value.append(yaw_arr[j]);
    tx_characteristic_string.writeValue(tx_estring_value.c_str());
  }
}
</code></pre>

Below plots the IMU data collected over a time > 5 seconds.
<p style="text-align:center;"><img src="..\assets\images\2\final_IMUdata_plot.png" width="550"/></p> 

Below is a snippet of the data received by bluetooth. The first value is the time in ms from the start of the data collection, the second value is pitch (deg), the third value is roll (deg), and the fourth value is yaw (deg).

<p style="text-align:center;"><img src="..\assets\images\2\final_data.png" width="300"/></p> 

## Stunt

I recorded the RC car accelerating in a "straight line." At some point, the car starts turning, possibly due to uneven power input to the motors, friction on the wheels, or change in the surface. It is very quick the accelerate, and it can withstand ramming into walls without breaking.

<p style="text-align:center;"><iframe width="420" height="600" src="https://www.youtube.com/embed/OjO9m1uLy8Q" allowfullscreen></iframe></p>

I also recorded the car doing a flip. This can be achieved by accelerating forwards until it reaches a high speed, and then immediately reversing. The RC car will decelerate quickly enough that its momentum will make it flip. If you don't hit reverse, the car will coast to a stop very quickly.

<p style="text-align:center;"><iframe width="420" height="600" src="https://www.youtube.com/embed/Bg7NVQi8a_g" allowfullscreen></iframe></p>