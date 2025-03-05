---
layout: project
title: Lab 4
description: Lab 4 Writeup
permalink: /lab4/
---

## Prelab

I planned to connect the motor drivers to pins A0, A1, A3, and A4, since A2 is already being used for the XSHUT on the To F sensor. 

Wire diagram:
<p style="text-align:center;"><img src="..\assets\images\4\wire_diagram.jpeg" width="950"/></p>

The motors draw a higher amount of current, especially when starting/stopping and under load (like friction on the wheels). So it makes sense to use a higher capacity battery for the motors. Additionally, microcontrollers like the Artemis need a stable voltage to function correctly. The current spikes from the motors would cause a voltage drop if the Artemis and motors shared the same battery. The voltage drops could cause the Artemis to behave unpredictably or reset. So it is better to have the Artemis and motors on separate batteries.

## First Motor Driver

<p style="text-align:center;"><img src="..\assets\images\4\first_motordriver.jpeg" width="950"/></p>

I soldered the wire connections for the first motor driver and solder them onto pine A0 and A1 on the Artemis Board. I tested the motor driver with a power supply, which I set to 3.7 V. I used the oscilloscope to show the output signal from the motor driver.

<p style="text-align:center;"><img src="..\assets\images\4\oscilloscope_setup.jpeg" width="950"/></p>

To test the motor driver, I ran it through a few different PWM duty cycles:

<pre><code class="language-cpp">#define PWM_PIN 0

void setup() {
  Serial.begin(9600);
  pinMode(PWM_PIN, OUTPUT);
  analogWrite(PWM_PIN, 0);
  Serial.println("Connected");

}

void loop() {
  analogWrite(PWM_PIN, 50);
  delay(2000);
  
  analogWrite(PWM_PIN, 100);
  delay(2000);

  analogWrite(PWM_PIN, 150);
  delay(2000);

  analogWrite(PWM_PIN, 200);
  delay(2000);

  analogWrite(PWM_PIN, 250);
  delay(2000);
}
</code></pre>

<p style="text-align:center;"><img src="..\assets\images\4\oscilloscope_display.jpeg" width="950"/></p>

<p style="text-align:center;"><iframe width="420" height="600" src="https://www.youtube.com/embed/NSpd1ee3_I0" allowfullscreen></iframe></p>

## Disassemble Car

I cut the wires off of the RC car manufacturer's PCB and removed it from the car. Now there's space for my components.

<p style="text-align:center;"><img src="..\assets\images\4\empty_car.jpeg" width="950"/></p>

## Motor

After testing the motor driver, I soldered it to one of the car's motors. I ran the code below to test the motors running in both directions. The motor is still being powered with the power supply.

<pre><code class="language-cpp">#define M1_FWD 1
#define M1_BWD 0

void setup() {
  pinMode(M1_FWD, OUTPUT);
  pinMode(M1_BWD, OUTPUT);
  analogWrite(M1_FWD, 0);
  analogWrite(M1_BWD, 0);

  Serial.begin(9600);
  Serial.println("Connected");
}

void loop() {
  // Motor drive forward for 3s
  analogWrite(M1_FWD, 255);
  delay(3000);
  
  // Stop for 1s
  analogWrite(M1_FWD, 0); 
  delay(2000);

  // Motor drive backward for 3s
  analogWrite(M1_BWD, 255);
  delay(3000);

  // Stop for 1s
  analogWrite(M1_BWD, 0); 
  delay(2000);
}
</code></pre>

<p style="text-align:center;"><iframe width="420" height="600" src="https://www.youtube.com/embed/x3PPknEzFoY" allowfullscreen></iframe></p>

## Battery-Powered Motors

I soldered the 650mAh battery connector onto the 850mAh connector so that I can charge it with the USB charger we already have to charge the 650 mAh bettery. The connector will also prevent me from plugging in the battery the wrong way, which is very likely if I'm tired and not paying attention. A matching female connector was left over from removing the manufacturer's PCB, so I wired that to the motors. 

<p style="text-align:center;"><img src="..\assets\images\4\battery.jpeg" width="950"/></p>

I then ran the motor powered with just the 850 mAh battery (instead of the power supply).

<p style="text-align:center;"><iframe width="420" height="600" src="https://www.youtube.com/embed/OlYvt8hlmX8" allowfullscreen></iframe></p>

## The Other Motor Driver

I wired the other motor driver the same way, onto pins A3 and A4 on the Artemis Board. I ran the code below to test both motors:

<pre><code class="language-cpp">#define ML_FWD 1
#define ML_BWD 0
#define MR_FWD 3
#define MR_BWD 4

void setup() {
  pinMode(ML_FWD, OUTPUT);
  pinMode(ML_BWD, OUTPUT);
  pinMode(MR_FWD, OUTPUT);
  pinMode(MR_BWD, OUTPUT);

  Serial.begin(9600);
  Serial.println("Connected");
}

void loop() {
  //Pause: allow time to disconnect battery or reset position
  delay(10000);

  analogWrite(ML_FWD, 100);
  analogWrite(MR_FWD, 100);
  delay(1200);

  analogWrite(ML_FWD, 0);
  analogWrite(MR_FWD, 0);
  delay(2000);

  analogWrite(ML_BWD, 100);
  analogWrite(MR_BWD, 100);
  delay(1200);

  analogWrite(MR_BWD, 0);
  analogWrite(ML_BWD, 0);
  delay(2000);
}
</code></pre>

<p style="text-align:center;"><iframe width="420" height="600" src="https://www.youtube.com/embed/AFMzUrTeUh4" allowfullscreen></iframe></p>

## Secure Components to Chassis

To get everything soldered together and so that I can have the robot drive on the ground, I secure the components to the car with tape. The artemis board and motor drivers are snuggly fit in the empty space that the manufacturer's PCB used to be. I designate the front of the car to be where the Artemis board is, and the rear is where the batteries sit. One ToF sensor is in the front, and the other one is in the rear.

<p style="text-align:center;"><img src="..\assets\images\4\car_diagram_1.jpeg" width="950"/></p>
<p style="text-align:center;"><img src="..\assets\images\4\car_diagram_2.jpeg" width="950"/></p>

The 650mAh battery connected to Artemis sits below the 850mAh battery in the battery box of the car.

<p style="text-align:center;"><img src="..\assets\images\4\car_diagram_3.jpeg" width="950"/></p>

## PWM Lower Limit

At very low PWM values, the car cannot move because there's not enough power to overcome static friction. For straight path, the lower limit PWM value I found is 30. For an on-axis turn, I found the lower limit PWM value is 130. This makes sense because there's much more friction on the wheels when turning on-axis, especially because the car is somewhat rectangular.

<p style="text-align:center;"><iframe width="420" height="600" src="https://www.youtube.com/embed/JglBm9aBzk8" allowfullscreen></iframe></p>

<p style="text-align:center;"><iframe width="420" height="600" src="https://www.youtube.com/embed/gBa6KJq1D6k" allowfullscreen></iframe></p>

<pre><code class="language-cpp">void driveStraight() {
  analogWrite(ML_FWD, 30);
  analogWrite(MR_FWD, 30);
  delay(1200);

  analogWrite(ML_FWD, 0);
  analogWrite(MR_FWD, 0);
  delay(2000);

  analogWrite(ML_BWD, 30);
  analogWrite(MR_BWD, 30);
  delay(1200);

  analogWrite(MR_BWD, 0);
  analogWrite(ML_BWD, 0);
  delay(2000);
}

void onaxisTurn(){
  analogWrite(ML_FWD, 130);
  analogWrite(MR_BWD, 130);
  delay(5000);

  analogWrite(ML_FWD, 0);
  analogWrite(MR_BWD, 0);
  delay(2000);

  analogWrite(ML_BWD, 130);
  analogWrite(MR_FWD, 130);
  delay(5000);

  analogWrite(ML_BWD, 0);
  analogWrite(MR_FWD, 0);
  delay(2000);
}
</code></pre>

## Calibration for Straightness

I ran the car at PWM value of 100 and manually calibrated by multiplying a calibration factor to the PWM value.

<pre><code class="language-cpp">#define ML_FWD 1
#define ML_BWD 0
#define MR_FWD 3
#define MR_BWD 4

// Calibration Factors:
#define ML_CF 1
#define MR_CF 0.88

void setup() {
  pinMode(ML_FWD, OUTPUT);
  pinMode(ML_BWD, OUTPUT);
  pinMode(MR_FWD, OUTPUT);
  pinMode(MR_BWD, OUTPUT);

  Serial.begin(9600);
  Serial.println("Connected");
}

void loop() {
  //Pause: allow time to disconnect battery or reset position
  delay(15000);

  analogWrite(ML_FWD, 100*ML_CF);
  analogWrite(MR_FWD, 100*MR_CF);
  delay(1000);

  analogWrite(ML_FWD, 0);
  analogWrite(MR_FWD, 0);
}
</code></pre>

With a calibration factor of 0.88 on the right motor, I was able to get the car to drive a little more than 6 feet and deviate less than one inch from the straight path. 

<p style="text-align:center;"><iframe width="420" height="600" src="https://www.youtube.com/embed/zNwIZK15-4Y" allowfullscreen></iframe></p>

## Open Loop

In the video below, I demonstrate untethered, open loop control of the car by driving the car in a "square." This entails driving straight for a bit, followed by a roughly 90 degree turn, looped infinitely.

<pre><code class="language-cpp">#define ML_FWD 1
#define ML_BWD 0
#define MR_FWD 3
#define MR_BWD 4

#define ML_CF 1
#define MR_CF 0.9

void setup() {
  pinMode(ML_FWD, OUTPUT);
  pinMode(ML_BWD, OUTPUT);
  pinMode(MR_FWD, OUTPUT);
  pinMode(MR_BWD, OUTPUT);

  analogWrite(ML_FWD, 0);
  analogWrite(ML_BWD, 0);
  analogWrite(MR_FWD, 0);
  analogWrite(MR_BWD, 0);

  Serial.begin(9600);
  Serial.println("Connected");
  
  //Pause: allow time to disconnect battery or reset position
  delay(10000);
}

void loop() {
  // Drive in square
  driveStraight();
  turn90deg();
}

void driveStraight() {
  analogWrite(ML_FWD, 90*ML_CF);
  analogWrite(MR_FWD, 90*MR_CF);
  delay(800);

  analogWrite(ML_FWD, 0);
  analogWrite(MR_FWD, 0);
  delay(1000);
}

void turn90deg(){
  analogWrite(ML_FWD, 140);
  analogWrite(MR_BWD, 140);
  delay(360);

  analogWrite(ML_FWD, 0);
  analogWrite(MR_BWD, 0);
  delay(1000);
}
</code></pre>

<p style="text-align:center;"><iframe width="420" height="600" src="https://www.youtube.com/embed/_N9wVkbTpUo" allowfullscreen></iframe></p>