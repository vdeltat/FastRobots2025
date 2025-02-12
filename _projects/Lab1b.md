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

<pre><code class="language-cpp">def notif_handler(uuid, data):
    notif = ble.bytearray_to_string(data)
    print("Current time: " + notif + " ms\n")
</code></pre>

## Task 5

I wrote a five second loop that sends the current time to my laptop that is recieved and processed by the notification handler.

<pre><code class="language-cpp">case LOOP_TIME:
    {
    unsigned long T0 = millis();

    int count = 0;
    while (millis() - T0 < 5000) { //for 5 sec
        
        unsigned long t = millis();  // Get current time in milliseconds
    
        char time[20];
        sprintf(time, "Sample %d: %lu", count, t);

        tx_estring_value.clear();
        tx_estring_value.append(time);
        tx_characteristic_string.writeValue(tx_estring_value.c_str());
        //Serial.println(tx_estring_value.c_str());
        count++;
    }

    Serial.print("Sent time data for 5 seconds. Total count: ");
    Serial.println(count-1);

    break;
    }
</code></pre>

125 data points were sent, so the effective data transfer rate is 25 data points a second.

<p style="text-align:center;"><img src="..\assets\images\1b\1b_5.png" width="550"/></p>
<p style="text-align:center;"><img src="..\assets\images\1b\1b_5_2.png" width="500"/></p>

## Task 6

I created a global array with a size of 1000 to store the time stamps, defined under the global variables at the top of the code. 

<pre><code class="language-cpp">#define MAX_TIME_SAMPLES 100
unsigned long time_stamps[MAX_TIME_SAMPLES];
float temps[MAX_TIME_SAMPLES];
</code></pre>

The loop in the command "SEND_TIME_DATA" fills the array with data until it is full, up to five seconds. Then another loop sends the data points in the array one at a time to my laptop.

<pre><code class="language-cpp">case SEND_TIME_DATA:
    {
    unsigned long T0 = millis();
    int index = 0; 

    while (millis() - T0 < 5000) {  // Run for 5 seconds
        if (index < MAX_TIME_SAMPLES) {  
            time_stamps[index] = millis(); 
            index++;
        } else {
            Serial.println("Array full.");
            break;
        }
    }
        
    //Send back the array
    for (int i = 0; i < index; i++) {
        char time[20];
        sprintf(time, "Sample %d: %lu", i+1, time_stamps[i]);

        tx_estring_value.clear();
        tx_estring_value.append(time);
        tx_characteristic_string.writeValue(tx_estring_value.c_str());

        Serial.println(tx_estring_value.c_str());
    }

    Serial.println("All timestamps sent.");
    break;
    }
</code></pre>

The array was full after 34 ms, so the effective data collection rate is 29,411 data points a second.

<p style="text-align:center;"><img src="..\assets\images\1b\1b_6.png" width="550"/></p>

## Task 7

Using the same method as Task 6, temperature readings are sent to the laptop in addition to time data, through the command GET_TEMP_READINGS, which loops through both the time and temperature arrays. I added to the notification handler so that it is able to parse the strings and populate both time and temperature data into two lists.

<pre><code class="language-cpp">time = []
temp = []

def notif_handler(uuid, data):
    notif = ble.bytearray_to_string(data)

    if("|" in notif):
        split = notif.split(" | ")
        print("Sample " + split[0] + ": " + split[1] + " ms is the current time and " + split[2] + " deg F is the current temp.")

        global time
        global temp
        time.append(float(split[1]))
        temp.append(float(split[2]))
    else:
        print("Current time: " + notif + " ms\n")
</code></pre>

The array was full after 343 ms, so the effective data transfer rate is 2,915 data points a second.

<p style="text-align:center;"><img src="..\assets\images\1b\1b_7.png" width="700"/></p>

## Task 8
Discuss the differences between these two methods, the advantages and disadvantages of both and the potential scenarios that you might choose one method over the other. How “quickly” can the second method record data? The Artemis board has 384 kB of RAM. Approximately how much data can you store to send without running out of memory?

Task 5's method allows you to access the data faster in real-time than task 6's method, but it slows the data collection frequency because it is limited by the speed it can send the data over bluetooth. Task 6's method can collect the data faster or at a higher frequency, but has a limited amount of data that can be stored at a time. However this can be mitigated by collecting and sending batches of data every so often.

A float and unsigned long are each 4 bytes of data. Given the Artemis board has 384 kB of RAM, the Artemis would be able to store 96,000 data points. If it was storing both the time and temperature arrays, you would be about to store 48,000 measurements. Some of the RAM is used for strings, Bluetooth libraries, and executing code, so actually there would be less data points that can be stored. Assuming 75% of RAM is left, that would be 72000 data points, or 36000 temperature measurements. This would mean the Artemis board can store about 12 seconds of temperature measurements can be stored at a time. 

## Additional Task 1: Effective Data Rate And Overhead

I wrote a function to get the data rate and overhead of a message sent from the computer and reply received from the Artemis board.

<pre><code class="language-cpp">import time
import sys
def get_receiveRate(message):
    ble.send_command(CMD.ECHO, message)
    start = time.time()
    received = ble.receive_string(ble.uuid['RX_STRING'])
    end = time.time()
    deltat = end - start
    bytesize = sys.getsizeof(received)
    rate = bytesize/deltat
    print("\nReceived: "+ received)
    print(str(bytesize) + " bytes")
    print(str(rate) + " bytes/s")

    overhead=bytesize-len(message.encode())
    return rate, overhead
</code></pre>

The data rate for 5-byte reply is 387 bytes/s and the data rate for a 120-byte reply is 1311 bytes/s. The overhead was 41 bytes for both replies.

<p style="text-align:center;"><img src="..\assets\images\1b\1b_extra1.png" width="700"/></p>

The size of the reply does not change the amount of overhead, but the larger the size of the reply, the higher the data rate.
<p style="text-align:center;"><img src="..\assets\images\1b\1b_extra1_plot_overhead.png" width="700"/></p>
<p style="text-align:center;"><img src="..\assets\images\1b\1b_extra1_plot_rate.png" width="700"/></p>

## Additional Task 2: Reliability

SEND_TIME_DATA sent data at a higher data rate than GET_TEMP_READINGS, but the computer was able to read all the data published from the Artemis boards for both commands. Data transfer is reliable even for transmitting data at high speeds.