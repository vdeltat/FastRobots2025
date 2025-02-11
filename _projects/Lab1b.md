---
layout: project
title: Lab 1B
description: Lab 1B Writeup
permalink: /lab1b/
---

Objective: Set up bluetooth connection with the Artemis Board and be able to receive timestamped messages from board.

## Configuration

I setup the bluetooth connection, changing the MAC Address in connections.yml to the MAC Address of the Artemis Board: c0:81:55:22:a3:64. I also generated the unique UUID, **e3ad7b8e-2664-4c72-954c-22bb7d2e4c55**, using uuid64() in python, and defined it in ble_arduino.ino and connections.yaml. 

<p style="text-align:center;"><img src="..\assets\images\1b\1b_demo.png" width="500"/></p>

## Task 1

Send a string value from the computer to the Artemis board using the ECHO command. The computer should then receive and print an augmented string.

<p style="text-align:center;"><img src="..\assets\images\1b\1b_1.png" width="950"/></p>

## Task 2

Send three floats to the Artemis board using the SEND_THREE_FLOATS command and extract the three float values in the Arduino sketch.

<p style="text-align:center;"><img src="..\assets\images\1b\1b_2.png" width="500"/></p>

## Task 3

Add a command GET_TIME_MILLIS which makes the robot reply write a string such as “T:123456” to the string characteristic.

<pre><code class="language-cpp">case GET_TIME_MILLIS:
    unsigned long t;
    t = millis(); // current time in ms
    char time[20];
    sprintf(time, "T:%lu", t);
    
    tx_estring_value.clear();
    tx_estring_value.append(time);
    tx_characteristic_string.writeValue(tx_estring_value.c_str());
    
    Serial.println(tx_estring_value.c_str());
    break;
</code></pre>

## Task 4

Setup a notification handler in Python to receive the string value (the BLEStringCharactersitic in Arduino) from the Artemis board. In the callback function, extract the time from the string.

## Task 5

Write a loop that gets the current time in milliseconds and sends it to your laptop to be received and processed by the notification handler. Collect these values for a few seconds and use the time stamps to determine how fast messages can be sent. What is the effective data transfer rate of this method?

<p style="text-align:center;"><img src="..\assets\images\1b\1b_5.png" width="550"/></p>
<p style="text-align:center;"><img src="..\assets\images\1b\1b_5_2.png" width="500"/></p>

## Task 6
Now create an array that can store time stamps. This array should be defined globally so that other functions can access it if need be. In the loop, rather than send each time stamp, place each time stamp into the array. (Note: you’ll need some extra logic to determine when your array is full so you don’t “over fill” the array.) Then add a command SEND_TIME_DATA which loops the array and sends each data point as a string to your laptop to be processed. (You can store these values in a list in python to determine if all the data was sent over.)

<p style="text-align:center;"><img src="..\assets\images\1b\1b_6.png" width="550"/></p>

## Task 7
Add a second array that is the same size as the time stamp array. Use this array to store temperature readings. Each element in both arrays should correspond, e.e., the first time stamp was recorded at the same time as the first temperature reading. Then add a command GET_TEMP_READINGS that loops through both arrays concurrently and sends each temperature reading with a time stamp. The notification handler should parse these strings and add populate the data into two lists.

<p style="text-align:center;"><img src="..\assets\images\1b\1b_7.png" width="700"/></p>

## Task 8
Discuss the differences between these two methods, the advantages and disadvantages of both and the potential scenarios that you might choose one method over the other. How “quickly” can the second method record data? The Artemis board has 384 kB of RAM. Approximately how much data can you store to send without running out of memory?

## Additional Task 1: Effective Data Rate And Overhead

Effective Data Rate And Overhead: Send a message from the computer and receive a reply from the Artemis board. Note the respective times for each event, calculate the data rate for 5-byte replies and 120-byte replies. Do many short packets introduce a lot of overhead? Do larger replies help to reduce overhead? You may also test additional reply sizes. Please include at least one plot to support your write-up.

## Additional Task 2: Reliability

Reliability: What happens when you send data at a higher rate from the robot to the computer? Does the computer read all the data published (without missing anything) from the Artemis board? Include your answer in the write-up.