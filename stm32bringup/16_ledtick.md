# I.6: The Tick

![It’s blue! It blinks! It’s the Tick!](https://warehouse.motd.org/wp-content/uploads/2020/11/tick.png)

In previous iteration, I made the user LED blink using an active delay
loop. I have two issues with this implementation:

- It’s hard to control the delay timing accurately

- Active loops are not cool

So I am going to call in some serious reinforcement, which means one of
the Arm Core Peripherals: the System Tick.

What the System Tick does is very similar to my active delay loop as
can be seen from the following pseudo-code.

```c
while( enabled) {
    if( --current_value == 0) {
        current_value = reload_value ;
        countflag = true ;
        if( interrupt_enabled)
            SysTick_Handler() ;
    }
}
```

It’s an auto decremented counter that reloads and sets a flag when
reaching zero. It can trigger a system interrupt if requested to. Its
default clock is the processor clock and can be switched to external
clock. Details can be found in the Programming Manual as this is part of
Arm Core.

## Code, build and test

I copy **blink.c** into **ledtick.c** to make the following
modifications.

- Expand the interrupt vector to make room for all 15 System Exceptions
from **Reset_Handler** to **SysTick_Handler**.

- Introduce the **SysTick** core peripheral to the compiler, using
pre-processor macroes to give the location of SysTick registers. As this
is a core peripheral, it’s in a different address space than the
peripherals we have seen so far.

- Start the **Reset_Handler** by initializing and enabling the System
Tick. The reload value register is initialized with a constant based on
the internal clock value divided by 8. As I want one tick per second and
the default internal clock is 8 MHz, SysTick will have to count one
million steps, from 999999 to 0. So the reload value is 999999.

- Create the **SysTick_Handler** interrupt routine which toggles GPIO
B1.

- Replace the previous active delay loop by a **cool** idle loop.
Instead of doing nothing actively, I instruct the processor to wait for
interrupt, which means it will suspend until waked up by the SysTick.
This is one way to lower power consumption, besides waiting for SysTick
interrupt is really the only thing left to do.

```c
/* Memory locations defined by linker script */
extern long __StackTop ;        /* &__StackTop points after end of stack */
void Reset_Handler( void) ;     /* Entry point for execution */

void SysTick_Handler( void) ;

/* Interrupt vector table:
 * 1  Stack Pointer reset value
 * 15 System Exceptions
 * NN Device specific Interrupts
 */
typedef void (*isr_p)( void) ;
isr_p const isr_vector[ 16] __attribute__((section(".isr_vector"))) = {
    (isr_p) &__StackTop,
/* System Exceptions */
    Reset_Handler,
    0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,  0,
    SysTick_Handler
} ;

#define SYSTICK             ((volatile long *) 0xE000E010)
#define SYSTICK_CSR         SYSTICK[ 0]
#define SYSTICK_RVR         SYSTICK[ 1]
#define SYSTICK_CVR         SYSTICK[ 2]

#define RCC                 ((volatile long *) 0x40021000)
#define RCC_AHBENR          RCC[ 5]
#define RCC_AHBENR_IOPBEN   0x00040000  /*  18: I/O port B clock enable */

#define GPIOB               ((volatile long *) 0x48000400)
#define GPIOB_MODER         GPIOB[ 0]
#define GPIOB_ODR           GPIOB[ 5]

void Reset_Handler( void) {
/* By default SYSCLK == HSI [8MHZ] */

/* SYSTICK */
    SYSTICK_RVR = 1000000 - 1 ;     /* HBA / 8 */
    SYSTICK_CVR = 0 ;
    SYSTICK_CSR = 3 ;               /* HBA / 8, Interrupt ON, Enable */
    /* SysTick_Handler will execute every 1s from now on */

/* User LED ON */
    RCC_AHBENR |= RCC_AHBENR_IOPBEN ;   /* Enable IOPB periph */
    GPIOB_MODER |= 1 << (1 * 2) ;       /* PB1 Output [01], over default 00 */
    /* OTYPER Push-Pull by default */
    /* PB1 output default LOW at reset */

    for( ;;)
        __asm( "WFI") ; /* Wait for interrupt */
}

void SysTick_Handler( void) {
    GPIOB_ODR ^= 1 << 1 ;   /* Toggle PB1 (User LED) */
}
```

I didn’t initialize the GPIO B before enabling the SysTick as I have a
whole second before the first interrupt will tick in.

Build is straightforward.

```
$ make ledtick.hex
ledtick.elf
   text    data     bss     dec     hex filename
    148       0       0     148      94 ledtick.elf
ledtick.hex
rm ledtick.o ledtick.elf
```

If we compare with blink.hex, 56 bytes of the 80 bytes code increase are
due to the expansion of the interrupt vector.

Once flashed in the board we can see the LED changes state every second.

## Checkpoint

I now have the foundation for timing and a first taste of shifting
execution between a main loop and an interrupt routine.

Code size has been growing steadily since the first bootstrap. On the
other hand, except for the stack, I have not used RAM memory so far.

|            | text | data  | bss   |
|------------|-----:|:-----:|:-----:|
|**boot**    | 10	| 0		| 0     |
|**ledon**   | 40	| 0		| 0     |
|**blink**   | 68	| 0		| 0     |
|**ledtick** | 148	| 0		| 0     |

[Next](https://warehouse.motd.org/?page_id=273) I will focus on RAM
initialization.

___
© 2020-2021 Renaud Fivet
