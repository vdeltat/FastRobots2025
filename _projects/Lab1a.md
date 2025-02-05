---
layout: project
title: Lab 1A
description: Lab 1A Writeup
permalink: /lab1a/
---

Objective: Familiarize myself with Arduino IDE and the SparkFun RedBoard Artemis Nano board. 

<h3>Task 1</h3>

Plugged in the Artemis board using a USB to my computer and selected the correct Board and Port in the ArduinoIDE.

<p style="text-align:center;"><img src="\assets\images\1a\Part1.png" width="950"/></p>

<h3>Task 2</h3>

File -> Examples -> 01.Basics: Blink - blinks the LED on the board.

<p style="text-align:center;"><iframe width="420" height="315" src="https://youtube.com/shorts/weIj_8gWCqY?si=yzrnF_VWLKHQ62QC"></iframe></p>

<h3>Task 3</h3>
File -> Examples -> Apollo3: Example4_Serial. I tested the serial communication between the board and the computer to send and receive information, which is useful for debugging. For it to work, the serial monitor has to match the baud rate set in the code: 115200.

<p style="text-align:center;"><img src="\assets\images\1a\Part3.png" width="950"/></p>

<h3>Task 4</h3>
File -> Examples -> Apollo3: Example2_analogRead. Used this file to test the temperature sensor on the Artemis board. I wrapped my hand, which is warmer than room temperature, around the artemis board, heating it up. You can see the temperature reading rise in the video.

<p style="text-align:center;"><iframe width="420" height="315" src="https://youtube.com/shorts/hHAIVsQY7iU?si=isEDf4ue2PI6rOWf"></iframe></p>

<h3>Task 5</h3>
In File -> Examples -> PDM: Example1_MicrophoneOutput. This file is used to test out the microphone. It outputs the highest frequency to the serial monitor.

<p style="text-align:center;"><iframe width="420" height="315" src="https://youtube.com/shorts/csPM9cUji4Q?si=iwdIs7xVxpUIaXPH"></iframe></p>

<h3>Additional task 1</h3>
Program the board to blink an LED when you play a musical “C” note over the speaker, and off otherwise. Use your phone, computer, or similar to generate the sound. If you’re having fun you could even combine the microphone and the Serial output to generate an electronic tuner.

<p style="text-align:center;"><iframe width="420" height="315" src="https://youtube.com/shorts/Z4TAi1dXcYM?si=Xfcv2tU_vs3fqCHm"></iframe></p>