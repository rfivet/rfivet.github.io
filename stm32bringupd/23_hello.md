# 2.3 Hello There!

Looking for a “_you had me at hello_” moment? Let’s see how serial
transmission works for you.

## Implementation

I make a copy of **board.c** into **usart1tx.c** to add support for the USART1
peripheral.

In order to make a first transmission, the peripherals have to be
initialized. As the TX/RX of USART1 are mapped on pin PA9 and PA10, we
need to configure GPIOA first.

- GPIOA needs to be enabled via RCC AHB Enable Register as GPIOs are on
AHB bus.

- PA9 and PA10 set to alternate mode.

- Alternate function USART1 selected for PA9 and PA10.

Then USART1 can be configured:

- USART1 enabled via RCC APB2 Enable Register as USARTs are on the APB
bus.

- Baud rate set to 9600 bauds.

- USART itself and transmission needs to be enabled via the Control
Register (CR1).

By default the transmission format is 8N1: 8 bit data, no parity and 1
stop bit.

```c
/* USART1 9600 8N1 */
    RCC_AHBENR |= RCC_AHBENR_IOP( A) ;  /* Enable GPIOA periph */
    GPIOA[ MODER] |= 0x0A << (9 * 2) ;  /* PA9-10 ALT 10, over default 00 */
    GPIOA[ AFRH] |= 0x110 ;             /* PA9-10 AF1 0001, over default 0000 */
    RCC_APB2ENR |= RCC_APB2ENR_USART1EN ;
    USART1[ BRR] = 8000000 / 9600 ;     /* PCLK [8MHz] */
    USART1[ CR1] |= USART_CR1_UE | USART_CR1_TE ;   /* Enable USART & Tx */
```

Sending data is done by writing in the Transmission Data Register (TDR).
To check if it is ready for transmission you must check the state of the
TX Empty (TXE) bit in the Interrupt & Status Register (ISR).

I write a basic `kputc()` function that does busy waiting if the **TDR** is
not empty and insures that LF are mapped to CR LF. The ‘k’ in kputc
refer to ‘kernel’, as kputc is a low level function that will be used
mostly for debugging. With the busy wait and the recursive code this
implementation is definitively not optimal, but it’s functional and
that’s what matter most at this stage.

```c
void kputc( unsigned char c) {
    static unsigned char lastc ;

    if( c == '\n' && lastc != '\r')
        kputc( '\r') ;

/* Active wait while transmit register is full */
    while( (USART1[ ISR] & USART_ISR_TXE) == 0) ;

    USART1[ TDR] = c ;
    lastc = c ;
}
```

The high level C function I need for this simple test is `puts()`. I make
my own implementation but I keep the same declaration as the standard
header that come with the C compiler.

Finally I use a standard C implementation for **hello.c**.

```c
/* hello.c -- hello there */
#include <stdio.h>
#include <stdlib.h>

int main( void) {
    puts( "hello, world") ;
    return EXIT_SUCCESS ;
}
```

## Build

To build I update the software composition in **Makefile** by adding a new
`SRCS` line.

`SRCS = startup.c usart1tx.c hello.c`

Calling make, I can see that there is now some variable in **BSS** section
of the RAM. It is `lastchar` local to `kputc()`. Because of word alignment
**BSS** occupies 4 bytes.

```
$ make
f030f4.elf
   text    data     bss     dec     hex filename
    413       0       4     417     1a1 f030f4.elf
f030f4.hex
f030f4.bin
```

## Testing

After flashing the board with the new executable, I place back the
**BOOT0** jumper and press the reset button, the board user LED blinks
as usual but I can see the RX LED on the USB to UART adapter flash
briefly when I release the reset button.

On Windows PC, if I use PuTTY or Arduino IDE to open **COM4** at 9600
baud, every time I press and release the reset button I can see ‘hello,
world’ displayed on a new line in the terminal window.

On Linux, when I plug in the USB to UART adapter, it enumerates as
**/dev/ttyUSB0**, so it is compatible with the USB driver for serial
ports. If I try to open it with Arduino IDE, I get an error message as I
need to belong to **dialout** group to open that TTY for reading and
writing.

`sudo usermod -a -G dialout $USER`

Once added to **dialout**, I can open **/dev/ttyUSB0** at 9600 baud in Arduino
IDE, each time I press and release the board RESET button, I can see
‘hello, world’ displayed on a new line in the Serial Monitor window.

## Checkpoint

I have now a functional serial transmission channel through **USART1**. I
have only a first implementation for `puts()`, but I will add support for
other stdio functions when needed.

[Next]( 24_stm32flash) I will switch to a tool for flashing over serial
connection that works on both Windows and Linux.

___
© 2020-2021 Renaud Fivet
