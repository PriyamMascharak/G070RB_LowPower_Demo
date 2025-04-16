# G070RB_LowPower_Demo

(Project Long Form Report: https://github.com/PriyamMascharak/G070RB_LowPower_Demo/blob/main/G070RB_LowPower_Demo_Report.pdf)
(Schematic: https://github.com/PriyamMascharak/G070RB_LowPower_Demo/blob/main/demo_Schematic.pdf)

This is a mini project to demonstrate the **Low Power Capabilities** of a **STM32G070RB**. This is essentially an interview assignment project I was given. This is the problem statement:-

## The Problem Statement

**Q Switch & LED Blinking with STM32G070RB**:
There is a tactile switch (only one switch) and LED connected to the microcontroller in a
**NUCLEO-G070RB** board. The LED is off initially. Depending on switch press, the LED blinks in following way.

1st Switch press: LED blinks at frequency of **0.5 Hz**.
2nd Switch press: LED blinks at frequency of **1 Hz**.
3rd Switch press: LED blinks at frequency of **2 Hz**.
4th Switch press: LED turns off.
5th Switch press: considered 1st switch press and the LED blinking cycle repeats.

Create circuit and code for target microcontroller. Assume relevant functions. Also,
optimize for power consumption. Use micro-controllers **low power modes** to achieve it.


### Assumptions
- Assumed that we are free to use any pin for the led to maximize low power performance
- Accuracy is not the priority for this blinking led output

### Choices
- Mostly used in-built **STHAL macros** and functions instead of direct register manipulation to increase readability and portability.
- More power could have been saved by moving the vector table to the **SRAM** and disabling flash all together but that seemed like a bit overkill considering we are already using the low-power-regulator
- used a little trick where i use rising inturrupt on the button instead of falling to make it register only one time per press(No matter how long the press is) 

## Program Flow:

### 1) Initialization

- Initialize **Flash Interface** and **Systick**.
- Configure System Clock by using the **High Speed Internal(HSI)** oscillator to complete initialization quickly
- We setup the GPIO pin **PC13** in **Interrupt Rising Mode**.
- We setup **TIM3** with a 32kHz clock source in mind an enable PWM in channel 1 with the channel output in **PA6**.
- We setup **TIM6** with a 32Khz clock source in mind with a period of **200ms**
- We enable the power interface clock by using the macro **__HAL_RCC_PWR_CLK_ENABLE()** ;
- We prepare to enter Low Power mode by entering **LOW_POWER_CLK_CONFIG()** where we
  - Initialize Typedef structs to
  	- Initialize **LSI** 
  	- Disable **HSI**
  	- Update **SysClk** source
  - Change the **Voltage Scaling** to **Scale 2** 
  - Initialize LSI and wait for LSI turn on
  - Setup LSI as **SYSCLK**
  - Disable HSI
  - Disable **PLL**
- We then return to the main function to Suspend the **SYSTICK Interrupt**
- Enable **Sleep on Exit**
- Enter **Low Power Sleep Mode** by using the **WFI** instruction. Using the HAL function also enables the Low Power Run mode for us before going to the Low-Power Sleep Mode

### 2) Running

- After Entering the Sleep mode the MCU listens for interrupts 
- Once an interrupt is triggered by the PC13 button, the **NVIC** calls the **EXTI callback function** and starts the CPU.
- Inside the callback function we start **TIM6**
- TIM6 has a 200ms period after which it fires another interrupt.
- In the **TIM6 Period Elapsed Callback** we first stop the Timer6 and Check if the GPIO PC13 is still set
- If it is set we register it as a button press.
- We go into the switch statement and check the mode. If it is **Mode zero** we Start the timer. The default period is set to 499 with equates to an LED blink of 0.5Hz
- As more button presses are detected we update the period by using the **__HAL_TIM_SET_AUTORELOAD()** macro to update the TIM3 auto-reload register
- When we reach **Mode 3** we update the auto-reload register to 499 and then stop the TIM3 to stop the LED blinking 
- After we process the switch statement we exit the Interrupt and the Cortex Core goes back to sleep because we enabled **SleepOnExit()**.
