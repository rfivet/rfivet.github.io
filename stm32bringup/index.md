# STM32 Bringup

## Introduction
Getting started with a micro-controller usually means picking up a board,
an IDE, some RTOS or a set of libraries. Depending of your level of experience,
your budget and the solutions you select, the learning curve may be a steep
one and what you will learn can be very limited if you end up cornered in a
sandbox with no understanding of what’s going on under the hood.

Commercial solutions and mature open source projects are a must if you want to
develop products with some level of quality. Unfortunately their complexity is
high because they have to satisfy complex requirements. Their documentation
and source code when available are often hard to navigate, out of date or just
not addressing what you need to learn.

Starting from scratch, on the other hand, is not something often documented and
when it is, it is usually after the fact. So if you want to learn how to do it
you need to catch the opportunity to watch someone going through the steps and
explaining what’s going on.

I will try to capture here my own “STM32 bring up” journey using a step by step
approach, writing down the problems faced and decisions taken while evolving
simple projects.

## Part I: Bring it up!
I proceed by small incremental steps that are easy to reproduce and simple
enough to adapt to a variant of the micro-controller or a different board
layout.

- Pick up a [toolchain]( 11_toolchain), install it and check that it can
  build an executable.

- Write a minimal [bootstrap]( 12_bootstrap) for a target
  micro-controller and build a first executable.

- [Flash]( 13_flash) the first executable in an actual board and verify
  that it boots.

- Provide feedback by turning the [user LED ON](14_ledon) and making it
  [blink]( 15_blink).

- Use the System [Tick]( 16_ledtick) to handle the blinking.

- Insure that RAM memory is initialized as expected for a [C startup](
  17_cstartup).

- Structure the code according to the [three stages]( 18_3stages): boot,
  initialization and main execution.

- [Publish]( 19_publish) the code to a web git repository for further
  evolution.

## <a id="part2">Part II: Let's talk!</a>

It’s time to move to a more talkative interface so that the board not
only winks but also speaks. Again I will go through several steps to get
to a working asynchronous serial communication.

- [Validate]( 21_uart) the serial connection by wiring a board with an
  USB to UART adapter and using a Windows Flash loader application to
  read the chipset flash memory.

- Make sure that the code evolved so far works on the [board]( 22_board)
  with a serial connection.

- Say [hello]( 23_hello) as first transmission.

- Use [stm32flash]( 24_stm32flash) as flashing tool on both Windows and
  Linux.

- [Prototype]( 25_prototype) an application that tells how long the
  system has been running.

- Write a production version of [uptime]( 26_uptime) application.

- Bundle the standard C library output functions into an actual
  [library]( 27_library).

- [Configure]( 28_clocks) baud rate and clocks.

- Handle the transmission with [interrupt]( 29_interrupt).

## <a id="part3">Part III: Sensors! So hot! So wet!</a>

- Implement [DHT11]( 31_dht11) humidity and temperature sensor reading.

- [Investigate]( 32_errata) the quality of the DHT11 measurements.

- Use [DS18B20]( 33_ds18b20) digital thermometer for accurate
  temperature reading.

- Trigger [ADC]( 34_adcvnt) conversion to read the internal voltage and
  temperature sensors.

- [Calibrate]( 35_calibrate) the internal temperature sensor.

- [Update]( 36_update) toolchain to latest.

- Build for [In RAM Execution]( 37_inram).

- Read a [Resistor]( https://warehouse.motd.org/?page_id=1421) Value.

___
© 2020-2021 Renaud Fivet
