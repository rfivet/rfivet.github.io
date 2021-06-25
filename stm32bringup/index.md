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

- Pick up a [toolchain](11_toolchain), install it and check that it can build
an executable.
- Write a minimal [bootstrap](12_bootstrap) for a target micro-controller and
build a first executable.
- [Flash](https://warehouse.motd.org/?page_id=248) the first executable in an
actual board and verify that it boots.
- Provide feedback by turning the
[user LED ON](https://warehouse.motd.org/?page_id=303) and making it
[blink](https://warehouse.motd.org/?page_id=324).
- Use the System [Tick](https://warehouse.motd.org/?page_id=359) to handle the
blinking.
- Insure that RAM memory is initialized as expected for a
[C startup](https://warehouse.motd.org/?page_id=273).
- Structure the code according to the
[three stages](https://warehouse.motd.org/?page_id=412): boot, initialization
and main execution.
- [Publish](https://warehouse.motd.org/?page_id=433) the code to a web git
repository for further evolution.

## Part II: Let's talk!
It’s time to move to a more talkative interface so that the board not only
winks but also speaks. Again I will go through several steps to get to a
working asynchronous serial communication.

- [Validate](https://warehouse.motd.org/?page_id=484) the serial connection by
wiring a board with an USB to UART adapter and using a Windows Flash loader
application to read the chipset flash memory.
- Make sure that the code evolved so far works on the
[board](https://warehouse.motd.org/?page_id=514) with a serial connection.
- Say [hello](https://warehouse.motd.org/?page_id=546) as first transmission.
- Use [stm32flash](https://warehouse.motd.org/?page_id=578) as flashing tool
on both Windows and Linux.
- [Prototype](https://warehouse.motd.org/?page_id=612) an application that
tells how long the system has been running.
- Write a production version of
[uptime](https://warehouse.motd.org/?page_id=657) application.
- Bundle the standard C library output functions into an actual
[library](https://warehouse.motd.org/?page_id=696).
- [Configure](https://warehouse.motd.org/?page_id=725) baud rate and clocks.
- Handle the transmission with
[interrupt](https://warehouse.motd.org/?page_id=763).

## Part III: Sensors! So hot! So wet!
- Implement [DHT11](https://warehouse.motd.org/?page_id=798) humidity and
temperature sensor reading.
- [Investigate](https://warehouse.motd.org/?page_id=849) the quality of the
DHT11 measurements.
- Use [DS18B20](https://warehouse.motd.org/?page_id=908) digital thermometer
for accurate temperature reading.
- Trigger [ADC](https://warehouse.motd.org/?page_id=946) conversion to read the
internal voltage and temperature sensors.
- [Calibrate](https://warehouse.motd.org/?page_id=1035) the internal
temperature sensor.
- [Update](https://warehouse.motd.org/?page_id=1180) toolchain to latest.
- Build for [In RAM Execution](https://warehouse.motd.org/?page_id=1295).
- Read a [Resistor](https://warehouse.motd.org/?page_id=1421) Value.

___
© 2020-2021 Renaud Fivet
