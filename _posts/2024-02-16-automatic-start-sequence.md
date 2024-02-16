---
title: "Automatic Start Sequence"
date: 2024-02-16
categories: [Embedded System]
tags: [STM32, c++, Embedded System]
author: Luke
---

# Quick Background on Start Sequences

Before getting into the project, it is important to understand what I am automating. A sailing start sequence consists of a 3- or 5-minute countdown. This countdown will have a total of four signals, depending on the start sequence the timing of the signals will differ. For a 5-minute sequence, there will be signals at 5-4-1-GO. For a 3-minute sequence there will be signals at 3-2-1-GO. There are also sequences called rolling starts where the end of one start sequence is the start of another. The automation of this will include all different variations of the sequence so its applicable in all situations.

![Desktop View](/assets/img/AutomaticStartSequence/StartSequenceExample.gif){: width="500" height="250" }

# Introduction

I work as a sailing race coach in the summer, and I had difficulty running a start sequence while driving a coach boat and giving feedback. During a sequence I constantly need to be looking at my watch to keep track of the time and be ready to give a signal, this gets increasingly difficult when I am trying to give meaningful feedback. If the start sequence could be automated it would make my life as coach much easier, create a more reliable and repeatable sequence, and overall improve the quality of training. For the project I will be using a STM32 MCU to control a load switch connected to a horn. The I/O will utilize 3 seven segment displays to show time and modes, along with three buttons. It will be powered by 3 recycled 18650s in series to produce 12 volts.
I created a prototype of this idea last summer in a day to test it and see if it was as useful as I thought it would be. After using it for a day I was sold, however, the prototype I made had some unreliable connections that were made apparent by the rough conditions on a coach boat. To solve this problem, I decided to create this project as a PCB and learn some bare metal while I’m at it.

# 7-Segment Displays

## How they work

7-Segment displays work by letting the programmer address each of the 7-segments in each digit individually to display a numeric value from 0-9. There are two common types of internal wiring that 7-segments display come with: Common Anode and Common Cathode. This refers to the flow of current from digit pins to segment pins. I will be using Common Cathode types, which means that every segment is internally connected at the cathode side, which is connected to the digit pin. When the digit pin is pulled low and the segment pins are pulled high, the segments turn on. To not display a segment, it should be pulled low to match that of the digit pin so there is no potential difference between the pins. As you can see controlling 7-segment displays is all about voltage control, and voltage difference between the digit and segment pins.

In my case each digit has all its segments sharing a wire with every other digit, as a result. only one digit can be addressed at a time Meaning to say that each digit will have to constantly be activated, written to, and then deactivated in loop, do this fast enough and I can display things on more than one digit at a time.

## Code for driving displays

Before writing the drivers, I created a header file to hold all of the information necessary such as the pins and ports of the segments and digits as they are connected to the STM32. I also included function definitions.
To easily address each pin and port I created arrays indexing the digits and segments numerically as well as alphabetically. I also used an array of uint8_t arrays to hold which segments should be high and low for each number 0-9. This later allows me to use loops to address and write to each digit.

```c
// List of pins for each digit
static const uint16_t DIGIT_PINS[] = {DIGIT_1_PIN, DIGIT_2_PIN, DIGIT_3_PIN, DIGIT_4_PIN};

// List of ports for each digit
static GPIO_TypeDef *DIGIT_PORTS[] = {DIGIT_1_PORT, DIGIT_2_PORT, DIGIT_3_PORT, DIGIT_4_PORT};

// List of pins for each segment
static const uint16_t SEG_PINS[] = {SEG_A_PIN, SEG_B_PIN, SEG_C_PIN, SEG_D_PIN, SEG_E_PIN, SEG_F_PIN, SEG_G_PIN};

// List of ports for each digit
static  GPIO_TypeDef *SEG_PORTS[] = {SEG_A_PORT, SEG_B_PORT, SEG_C_PORT, SEG_D_PORT, SEG_E_PORT, SEG_F_PORT, SEG_G_PORT};

// List representing which segments should be to on to display numbers 0-9
uint8_t NUMS[10][7] = {
		{1,1,1,1,1,1,0}, // 0
		{0,1,1,0,0,0,0}, // 1
		{1,1,0,1,1,0,1}, // 2
		{1,1,1,1,0,0,1}, // 3
		{0,1,1,0,0,1,1}, // 4
		{1,0,1,1,0,1,1}, // 5
		{1,0,1,1,1,1,1}, // 6
		{1,1,1,0,0,0,0}, // 7
		{1,1,1,1,1,1,1}, // 8
		{1,1,1,1,0,1,1}};// 9
```

To select which digit to write to I wrote a function that pulls one digit high and the rest low by using the STM32 GPIO driver write pin function. This function takes advantage of the arrays I made previously to utilize a loop to go through all the digits efficiently. Note that I only wrote this driver to accommodate 4 digits, however, it can be easily modified to accommodate more if that is necessary in later projects.

```c
// Input: digit number 1-4
// Will enable the inputed digit and disable all other digits.
void selectDigit(uint32_t digit){
    for (int i = 0; i < 4; i++){
		// Enables inputed digit (pulls low)
		if (i == digit - 1){
			HAL_GPIO_WritePin(DIGIT_PORTS[i], DIGIT_PINS[i], 0);
		}
		// Disables unwanted digits (pulls high)
		else{
			HAL_GPIO_WritePin(DIGIT_PORTS[i], DIGIT_PINS[i], 1);
		}
	}
}
```

To write a number to the digit the segment pins are pulled either high or low depending on value in the NUMS array for the chosen number and segment number. Again the use of the previously made arrays makes this code very minimal when utilizing a for loop.

```c
// Input: integer from 0-9
// Displays a number on the selected digit by pulling all pins as they are
// represented in the NUMS list.
void writeNum(uint32_t num){
	// Runs once for each segment
	for (int i = 0; i < 7; i++){
		HAL_GPIO_WritePin(SEG_PORTS[i], SEG_PINS[i], NUMS[num][i]);
	}
}
```

## Using interrupts

If I were to place the 7-segment display logic in the main program loop it would appear choppy as other code was executed, as soon as there is a delay the screen would only be able to display one digit. To get around this and consistently switch between addressing each digit, I used an interrupt that was called every 3.75 milliseconds so that it would be separate from the main program loop. An interrupt works by taking the clock frequency of the oscillator in the MCU, dividing it, and then counting a certain number of cycles. These values are the pre scaler, and period counter or auto reload register (ARR) respectively. After it has counted the necessary number of cycles the interrupt will be triggered, and it will start over again.
I selected a pre-scaler value of 200 to divide the 16MHz clock signal into a 80 kHz one and calculated the counter period to fit 3.75 milliseconds.

$$
t = \frac{(ARR)(PSC)}{f}
$$
$$
3.75 ms = \frac{(ARR)(200)}{16 MHz}
$$
$$
ARR = 300
$$

After configuring the PSC and ARR values for the timer in the STM32 cube IDE and selecting TIM7 to be a global interrupt I was almost done. The STM32 cube IDE makes creating interrupts easy. They provide a call back function that is called upon every interrupt trigger and passes a reference to the TIM that triggered it. This allows multiple interrupts to be easily used and managed.

```c
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim){
}
```
