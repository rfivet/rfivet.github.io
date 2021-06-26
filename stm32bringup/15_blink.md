# I.5: Blinking user LED

Turning the user LED on shows that the board is alive, making it blink
shows that it has a pulse! A steady power LED and a pulsing user LED is
easy to interpret.

As I already manage to turn the LED on, making it blink is quite
straightforward.

## Code and build

I make a copy of **ledon.c** into **blink.c** and modify it as follow:

- Introduce the Output Data Register (**ODR**) of **GPIOB** peripheral to the
compiler as **GPIOB_ODR**. This is where the pin state is programmed.
- Modify the idle loop to toggle PB1 after a delay. The delay is coded
as an active wait loop on decrementing a counter.

```c
/* Memory locations defined by linker script */
extern long __StackTop ;        /* &__StackTop points after end of stack */
void Reset_Handler( void) ;     /* Entry point for execution */

/* Interrupt vector table:
 * 1  Stack Pointer reset value
 * 15 System Exceptions
 * NN Device specific Interrupts
 */
typedef void (*isr_p)( void) ;
isr_p const isr_vector[ 2] __attribute__((section(".isr_vector"))) = {
    (isr_p) &__StackTop,
/* System Exceptions */
    Reset_Handler
} ;

#define RCC                 ((volatile long *) 0x40021000)
#define RCC_AHBENR          RCC[ 5]
#define RCC_AHBENR_IOPBEN   0x00040000  /*  18: I/O port B clock enable */

#define GPIOB               ((volatile long *) 0x48000400)
#define GPIOB_MODER         GPIOB[ 0]
#define GPIOB_ODR           GPIOB[ 5]

void Reset_Handler( void) {
    int delay ;

/* User LED ON */
    RCC_AHBENR |= RCC_AHBENR_IOPBEN ;   /* Enable IOPB periph */
    GPIOB_MODER |= 1 << (1 * 2) ;       /* PB1 Output [01], over default 00 */
    /* OTYPER Push-Pull by default */
    /* PB1 output default LOW at reset */

/* User LED blink */
    for( ;;) {
        for( delay = 1000000 ; delay ; delay--) ;   /* delay between toggling */
        GPIOB_ODR ^= 1 << 1 ;                       /* toggle PB1 (User LED) */
    }
}
```

I set the value of the delay counter at one million. By default the
internal clock is set to 8MHz at reset, which means the delay will still
be less than one second.

```
$ make blink.hex
blink.elf
   text    data     bss     dec     hex filename
     68       0       0      68      44 blink.elf
blink.hex
rm blink.o blink.elf
```

## Test

[video](https://warehouse.motd.org/wp-content/uploads/2020/11/Blink.mp4)

As the video shows, the delay is roughly 600ms. I captured three on/off
transition in this three second video, looking through the frames gives
me a better estimation.

## Checkpoint

This is just a small increment on my previous step, but that’s iterative
development in a nutshell. Also I didn’t come with a reasonable value
for the delay counter at first, it’s easy to underestimate how fast
micro-controllers are.

[Next](16_ledtick), interrupt driven blinking, because active wait delay
is just not cool.

___
© 2020-2021 Renaud Fivet
