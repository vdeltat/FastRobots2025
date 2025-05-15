---
layout: project
title: Lab 9
description: Lab 9 Writeup
permalink: /lab9/
---

## Mapping

I started with taking distance measurements at every marked location in the static room. I would command the robot to turn to a certain angle using "START_ORI_PID" command (refer to lab 6) and then call "GET_MAPDATA" to collect the distance measurement from the front ToF sensor in that direction. I repeated this for increments of 25 degrees for the angle, collected distance measurements for each angle. I set a high timing budget since the car stays still while distance measurement is being collected. Retrospectively, I should have collected multiple distance measurements for each angle and averaged them.

<pre><code class="language-cpp">case GET_MAPDATA: {
  getMapData();
  break;
}

void getMapData(){
  int dist;
  while (!myICM.dataReady()) {
    delay(1);
  }
  getOrientation(); // gets yaw

  distanceSensor1.setDistanceModeLong();
  distanceSensor1.setTimingBudgetInMs(500);
  distanceSensor1.startRanging();
  while (!distanceSensor1.checkForDataReady()) {
      delay(1);
  }
  dist = distanceSensor1.getDistance();

  distanceSensor1.clearInterrupt();
  distanceSensor1.stopRanging();
  tx_estring_value.clear();       
  tx_estring_value.append(yaw);
  tx_estring_value.append(" | "); 
  tx_estring_value.append(dist);
  tx_characteristic_string.writeValue(tx_estring_value.c_str());
}

</code></pre>

After all the 15 measurements were collected, I sent the data over bluetooth.


<pre><code class="language-cpp">case SEND_MAP_DATA: {
  sendMappingData();
  break;
}

void sendMappingData() {
  for (int j = 0; j < angIndex; j++) { 
    tx_estring_value.clear();       
    tx_estring_value.append(map_angles[j]);
    tx_estring_value.append(" | "); 
    tx_estring_value.append(map_dists[j]);
    tx_characteristic_string.writeValue(tx_estring_value.c_str());
    delay(5);
  }
}
</code></pre>

## Creating Map

I took these measurements at 5 locations in the static room: the origin (0,0), (-3,-2), (0,3), (5,3), (5,-3), and saved them to their own csv files. To combine the data into a map, I converted all the distance vs. angle measurements and to cartesian coordinates using a series of transformation matrices. I first converted the distance measurements taken in mm to feet. The first transformation matrix (T1) takes into account the distance between the center of the robot and the center of the ToF sensor. The ToF sensor is centered in the y and z directions, and 3.25 inches from the center in the x direction, or 0.2708333 ft. The second transformation matrix (T2) takes into account the rotation from the angle, converting the robot’s frame of reference to the inertial coordinates. We use the standard rotation matrix about the z-axis, at point (xr,yr) relative to the origin. Lastly, we have the measurement vector (m) with distance d.

<p style="text-align:center;"><img src="..\assets\images\9\matrices.png" width="700"/></p>


<pre><code class="language-python">def transform(th, d, xr, yr):
    d = d*0.00328084 # convert mm to ft
    T1 = np.array([[1,0,0,0.2708333],
                    [0,1,0,0],
                    [0,0,1,0],
                    [0,0,0,1]])
    T2 = np.array([[np.cos(th),-np.sin(th),0,xr],
                    [np.sin(th),np.cos(th),0,yr],
                    [0,0,1,0],
                    [0,0,0,1]])
    meas = np.array([[d],
                      [0],
                      [0],
                      [1]])

    coords = np.matmul(T2,np.matmul(T1,meas))

    return coords
</code></pre>

## Plotting Map

I plotted the measurements taken at each location on a polar plot and in cartesion coordinates, shown below. 

<p style="text-align:center;">
  <img src="..\assets\images\9\pp0.png" width="450" />
  <img src="..\assets\images\9\cp0.png" width="450" /> 
</p>

<p style="text-align:center;">
  <img src="..\assets\images\9\pp1.png" width="450" />
  <img src="..\assets\images\9\cp1.png" width="450" /> 
</p>

<p style="text-align:center;">
  <img src="..\assets\images\9\pp2.png" width="450" />
  <img src="..\assets\images\9\cp2.png" width="450" /> 
</p>

<p style="text-align:center;">
  <img src="..\assets\images\9\pp3.png" width="450" />
  <img src="..\assets\images\9\cp3.png" width="450" /> 
</p>

<p style="text-align:center;">
  <img src="..\assets\images\9\pp4.png" width="450" />
  <img src="..\assets\images\9\cp4.png" width="450" /> 
</p>

# Final Map

Below is the combined plot of the points, and plotted with the actual map overlayed on top. There is a lot of noise in the measurements so the map is terrible. As I mentioned before, taking multiple distance measurements at each angle and averaging them would help with this. I also suspect that the ToF sensor might have been tilted a bit towards the ground for some of the measurements, resulting in inaccurate readings.

<p style="text-align:center;">
  <img src="..\assets\images\9\combined_plot.png" width="450" />
  <img src="..\assets\images\9\FinalCombinedMap.png" width="450" /> 
</p>







