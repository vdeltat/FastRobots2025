---
layout: project
title: Lab 3
description: Lab 3 Writeup
permalink: /lab3/
---

## Prelab

The default I2C address for the TOF sensor is 0x52 according to the datasheet. Since both sensors share the same default address, one of them needs to be reassigned a different I2C address. I wired the XSHUT pin of one sensor to A2, allowing me to temporarily disable it during startup, assign a new address, and then enable it again. This makes sure both sensors can operate simultaneously.

For placement, I intend to mount one ToF sensor at the front of the car and the other on the right side of the car. The front ToF sensor will be able to see oncoming obstacles, and the ToF sensor on the right side will be able to detect walls to avoid colliding with them, as well as helping with mapping the surroundings. However, the downsides are that the car won't be able to detect obstacles behind it, but if the car doesn't really back up and primarily maneuvers to drive forward in the direction it wants to go, it shouldn't be much of an issue.

<p style="text-align:center;"><img src="..\assets\images\3\Wire_diagram.jpeg" width="950"/></p>

## ToF Sensor Connections
Below shows the two ToF sensors and the IMU connected to the Artemis through the Qwiic Multiport. The Artemis is also being powered by the battery, which I soldered the connector onto.

<p style="text-align:center;"><img src="..\assets\images\3\all_sensors_wired.jpeg" width="950"/></p>

## I2C Address

I ran the example code: Examples->Apollo3->Wire->Example05_Wire_I2C.ino

After running the scan, the sensor appeared at address 0x29 instead of the expected 0x52. The 0x52 address listed in the datasheet includes the least significant bit (LSB) used for read/write operations in I2C communication. 0x52 = 0b101000, and  0x29 = 0b101001, so Arduino ignores the last bit and only returns the 7-bit address, which results in 0x29 (0x52 >> 1). 


<p style="text-align:center;"><img src="..\assets\images\3\I2C_scan.png" width="500"/></p>

## ToF Sensor Mode

The ToF sensor operates in three modes: Short, Medium, and Long. Each optimizes performance based on the expected range. Short mode has a maximum range of 1.3 meters, Medium - 3 meters, and Long - 4 meters.

Short mode provides greater robustness against noise and better data granularity, as the same bit resolution captures smaller distances more precisely. This makes it ideal for detecting nearby obstacles with high reliability. On the other hand, Long mode allows for a greater sensing range but can be affected by ambient light and reflections, potentially reducing accuracy.

Given that the car is going to be indoors in confined spaces, and obstacles will probably be relatively close since it is going fast, Short mode for both ToF sensors would be the best choice.

## Test ToF Sensor Short Mode

To evaluate the short mode, I collected 50 samples at distances from 100 mm to 1800 mm away from a wall in increments of 100 mm. 

<pre><code class="language-cpp">case TOF_GET_DISTANCE: {
  distanceSensor.setDistanceModeShort();

  for (int i = 0; i < DIST_SAMPLES; i++){

    distanceSensor.startRanging(); //Write configuration bytes to initiate measurement

    while (!distanceSensor.checkForDataReady()){
      delay(1);
    }
    
    distances[i] = distanceSensor.getDistance(); // collect distance data
    distanceSensor.clearInterrupt();
    
    distanceSensor.stopRanging();

    dist_times[i] = millis(); // get data time-stamp
  }

  for (int j = 0; j < DIST_SAMPLES; j++){
    tx_estring_value.clear();
    char time_str[20];
    sprintf(time_str, "%lu", dist_times[j]);
    tx_estring_value.append(time_str);
    tx_estring_value.append(" | ");
    tx_estring_value.append(distances[j]);
    tx_characteristic_string.writeValue(tx_estring_value.c_str());
  }

  break;
}
</code></pre>

The ToF Sensor is quite accurate and precise, even a bit beyond its maximum short mode range of 1.3 meters, although that is when the standard deviation starts increasing as expected.

<p style="text-align:center;"><img src="..\assets\images\3\ToF_Range.png" width="650"/></p>
<p style="text-align:center;"><img src="..\assets\images\3\ToF_Repeatability.png" width="650"/></p>
<p style="text-align:center;"><img src="..\assets\images\3\ToF_Accuracy.png" width="650"/></p>

The ranging times for decreases, surprisingly.
<p style="text-align:center;"><img src="..\assets\images\3\ToF_Times.png" width="650"/></p>

## 2 ToF Sensors

To use both ToF sensors simultaneously, I turned off the one sensors to modify the second sensor and give it a different I2C address, then re-enabled sensor 1.


<pre><code class="language-cpp">void setup(void)
{
  Wire.begin();

  Serial.begin(9600);
  Serial.println("VL53L1X Qwiic Test: Two ToFs");

  pinMode(SHUTDOWN_PIN, OUTPUT);
  digitalWrite(SHUTDOWN_PIN, LOW); // temp turn of sensor 1
  distanceSensor2.setI2CAddress(0x54);
  digitalWrite(SHUTDOWN_PIN, HIGH); // turn sensor on
</code></pre>

In the loop, I collect the sensor data if it is ready, for each individual sensor. If it is not ready, it will continue to loop through until it is, which allows it to run at full speed, instead of waiting for the sensor to be ready before proceeding through the loop.

<pre><code class="language-cpp">void loop(void)
{
  Serial.print("Time: ");
  Serial.print(millis());
  Serial.println(" ms");

  distanceSensor1.startRanging();
  distanceSensor2.startRanging();

  if(distanceSensor1.checkForDataReady()){ // next loop if not
    int dist1 = distanceSensor1.getDistance();
    distanceSensor1.clearInterrupt();
    distanceSensor1.stopRanging();
    Serial.print("ToF 1 Distance: ");
    Serial.print(dist1);
    Serial.println(" mm");
  }

  if(distanceSensor2.checkForDataReady()){
    int dist2 = distanceSensor2.getDistance();
    distanceSensor2.clearInterrupt();
    distanceSensor2.stopRanging();
    Serial.print("ToF 2 Distance: ");
    Serial.print(dist2);
    Serial.println(" mm");
  }
}
</code></pre>

A snapshot of results is shown below. Both sensors were able to collect data. As you can see, the Artemis loop runs faster at about 19 ms per loop (52 Hz), while the ToF sensors collect data about every 310 ms (3.2 Hz). Since the sensors run slower than the Artemis loop, they are the limiting factor.

<p style="text-align:center;"><img src="..\assets\images\3\Two_ToF.png" width="200"/></p>


## Combine IMU and ToF Sensor Data Collection

<p style="text-align:center;"><img src="..\assets\images\3\IMU_data1.png" width="650"/></p>
<p style="text-align:center;"><img src="..\assets\images\3\IMU_data2.png" width="650"/></p>
<p style="text-align:center;"><img src="..\assets\images\3\distance_data.png" width="650"/></p>


<pre><code class="language-cpp">while (central.connected()) {
  // Send data
  write_data();

  if (myICM.dataReady() && collect) {
    getIMUdata();
  }

  if (collect){
    distanceSensor1.startRanging();
    distanceSensor2.startRanging();

    while (!distanceSensor1.checkForDataReady()){
      delay(1);
    }
    while (!distanceSensor2.checkForDataReady()){
      delay(1); 
    }

    if (dist1_index >= SAMPLES){
      Serial.println("Dist1 Array full");
      continue;
    }
    if (dist2_index >= SAMPLES){
      Serial.println("Dist2 Array full");
      continue;
    }

    dist1_arr[dist1_index]=distanceSensor1.getDistance();
    distanceSensor1.clearInterrupt();
    distanceSensor1.stopRanging();
                
    dist2_arr[dist2_index]=distanceSensor2.getDistance();
    distanceSensor2.clearInterrupt();
    distanceSensor2.stopRanging();

    dist1_index++;
    dist2_index++;
  }
</code></pre>

## Additional Tasks (5000)

### IR-based Sensors

IR Triangulation Sensors determine distance by sending out an infrared pulse at a specific angle and measuring the angle of the reflected signal. These sensors are relatively accurate and insensitive to surface texture or color, making them useful for short-range applications. However, they are bulky, slow, and more expensive than simpler IR sensors.

Amplitude-Based IR Sensors work by comparing the strength of the emitted and received infrared signals. They are inexpensive and compact but only function over short distances and are really sensitive to ambient light, texture, and object color, which can affect the accuracy of measurements.

ToF Sensors like the ones we're using measure distance by timing how long it takes for an infrared laser pulse to travel to an object and return. They're largely unaffected by ambient light, texture, or color. You can get reliable measurements over longer distances but they tend to have a lower sample frequency and a bit higher cost.





