# 2.8: Baud Rate and Clocks

> “The time has come,” the walrus said, “to talk of many things: Of baud
  rates – and clocks – and quartz.”

> Les huit scaroles

One thing to consider in any kind of transmission is the speed, how fast
or how slowly you can transmit data. I have configured **USART1** at **9600
baud**, keeping the other settings at default value (**8N1**), so how fast is
that?

## A bit of theory

Let’s interpret asynchronous serial transmission, 9600 baud, 8 bits, no
parity, 1 stop bit.

- Serial transmission means transmission on one wire, with each bit sent
one after the other, usually low bit first.

- Asynchronous means there is no extra wire for a clock, so transmitter
and receiver must agree on a bit rate and a transmission pattern.

- Transmission pattern, **8N1** in my case, is composed of a start bit,
a data length (**8**), parity (Even, Odd or **N**one) and **1** or 2
stop bits.

- Because there is no common transmitted clock, the receiver
resynchronizes based on the start bit/stop bit framing of the data. It
samples the line at 16 times the frequency of the agreed clock to detect
the change in the wire state.

In my case, **8N1** means that, because of the framing pattern, for
every byte of data sent, there is one extra start bit and one extra stop
bit sent, it’s ten bits per byte of data. At 9600 bauds that means 960
bytes per second, fast enough to transmit every characters of a 80×24
terminal screen in two seconds.

## Baud rate accuracy

It sounds like a pretty robust transmission scheme, sampling at 16 times
the transmission clock isn’t call oversampling for nothing. Am I
overdoing something here or just compensating for something I missed?

The thing is, I didn’t program USART1 to transmit at 9600 baud. As my
default clock is 8MHz, I had to write in USART1 baud rate register a
value close to 8000000/9600 or 2500/3, 833 is close enough but my actual
transmission speed is closer to 9604, slightly faster than 9600 baud.

The error is small (4/10000) and the transmission works fine. Still
common baud rates are 300, 1200, 2400, 9600, 19200, 38400, 57600, 115200.
It would be better if my clock frequency was 6MHz or 12MHz if I
want to work at higher baud rate.

## Clocks

Looking at the clock tree in the datasheet can be intimidating, it’s
definitively about several clocks.

![Clock Tree]( img/28_clocktree.png)

The default configuration I have been using so far goes like this.

- HSI is the output of a 8MHz High Speed Internal RC oscillator.

- HSI is the default source of SYSCLK.

- HCLK, the clock of AHB domain, is SYSCLK divided by HPRE a pre-scaler
which default to 1.

- The system tick clock default configuration is HCLK/8.

- PCLK, the clock of APB domain, is HCLK divided by PPRE a pre-scaler
which default to 1.

- GPIO peripherals are on APB domain bus, they use PCLK.

- USART1 is also on APB domain bus, but its input clock can be selected,
PCLK is the default input.

From the peripherals point of view.

- SysTick Clock = HCLK/8 = SYSCLK/1/8 = HSI/1/8 = 8/1/8 MHz

- GPIOx Clock = PCLK = HCLK/1 = SYSCLK/1/1 = HSI/1/1 = 8/1/1 MHz

- USART1 Clock = PCLK = … = 8 MHz

As I want to have a clock frequency different than 8 MHz as input for
USART1, I can configure the Phase-Locked Loop (PLL) and switch SYSCLK to
take its input from the PLL instead of HSI.

The PLL output frequency must be in the range 16-48 MHz. As I am looking
for a frequency that can be divided by 3 to match most of the baud rate,
I will use 24 MHz.

- Select PLL input as HSI/2.

- Set PLLMUL to 6.

- Enable PLL and wait that it stabilizes.

- Select SYSCLK input as PLL.

- Wait for the switch to complete.

## Quartz

I can also activate the quartz if there is one soldered on the board.
It’s usually the case but specially for STM32F030F4 which has only 20
pins, a quartz less design that free up two GPIO pins can be a day
saver. Quartz value from 4 to 32 MHz are supported and most design use 8
MHz.

To set a 24 MHz clock with a 8 MHz High Speed External Oscillator (HSE):

- Enable HSE and wait that it stabilizes.

- Select HSE as input for the PLL with a pre divider of 2.

- Set PLLMUL to 6.

- Enable PLL and wait that it stabilizes.

- Select SYSCLK input as PLL.

- Wait for the switch to complete.

- Disable HSI.

I can use different values for the pre divider and post multiplier of
the PLL (/4, *12 or /1, *3 instead of /2, *6) but I want here to stay
aligned with the HSI/2 input selection when HSE quartz value is 8MHz.

## Implementation

I make a copy of **uplow.2.c** into **clocks.c** to make the changes.

I expand the board description part by adding `HSE`, `PLL` and `BAUD`
macro definitions. Based on those I can handle four clock
configurations: HSI, HSE, PLL HSI and PLL HSE.

```c
/* user LED ON when PA4 is high */
#define LED_IOP A
#define LED_PIN 4
#define LED_ON  1
/* 8MHz quartz, configure PLL at 24MHz */
#define HSE     8000000
#define PLL     6
#define BAUD    9600

#ifdef PLL
# ifdef HSE
#  define CLOCK HSE / 2 * PLL
# else /* HSI */
#  define CLOCK 8000000 / 2 * PLL
# endif
# if CLOCK < 16000000
# error PLL output below 16MHz
# endif
#else
# ifdef HSE
#  define CLOCK HSE
# else /* HSI */
#  define CLOCK 8000000
# endif
#endif

#if CLOCK > 48000000
# error clock frequency exceeds 48MHz
#endif

#if CLOCK % BAUD
# warning baud rate not accurate at that clock frequency
#endif
```

At compilation time there will be a check if the clock targeted is in
the supported range of the chipset and a warning given if the baud rate
generation is not accurate.

I expand the definition of the Reset and Clock Control (RCC) peripheral
to add the necessary bit fields.

```c
#define CAT( a, b) a##b
#define HEXA( a) CAT( 0x, a)
#define RCC                     ((volatile long *) 0x40021000)

#define RCC_CR          RCC[ 0]
#define RCC_CR_HSION    0x00000001  /*  1: Internal High Speed clock enable */
#define RCC_CR_HSEON    0x00010000  /* 16: External High Speed clock enable */
#define RCC_CR_HSERDY   0x00020000  /* 17: External High Speed clock ready flag$
#define RCC_CR_PLLON    0x01000000  /* 24: PLL enable */
#define RCC_CR_PLLRDY   0x02000000  /* 25: PLL clock ready flag */

#define RCC_CFGR            RCC[ 1]
#define RCC_CFGR_SW_MSK     0x00000003  /* 1-0: System clock SWitch Mask */
#define RCC_CFGR_SW_HSE     0x00000001  /* 1-0: Switch to HSE as system clock */
#define RCC_CFGR_SW_PLL     0x00000002  /* 1-0: Switch to PLL as system clock */
#define RCC_CFGR_SWS_MSK    0x0000000C  /* 3-2: System clock SWitch Status Mask$
#define RCC_CFGR_SWS_HSE    0x00000004  /* 3-2: HSE used as system clock */
#define RCC_CFGR_SWS_PLL    0x00000008  /* 3-2: PLL used as system clock */
#define RCC_CFGR_PLLSRC         0x00010000
#define RCC_CFGR_PLLSRC_HSI     0x00000000      /* HSI / 2 */
#define RCC_CFGR_PLLSRC_HSE     0x00010000      /* HSE */
#define RCC_CFGR_PLLXTPRE       0x00020000
#define RCC_CFGR_PLLXTPRE_DIV1  0x00000000  /* HSE */
#define RCC_CFGR_PLLXTPRE_DIV2  0x00020000  /* HSE / 2 */
#define RCC_CFGR_PLLMUL_MSK     (0x00F << 18)
#define RCC_CFGR_PLLMUL( v)     ((v - 2) << 18)

#define RCC_AHBENR              RCC[ 5]
#define RCC_AHBENR_IOP( h)      (1 << (17 + HEXA( h) - 0xA))

#define RCC_APB2ENR             RCC[ 6]
#define RCC_APB2ENR_USART1EN    0x00004000  /* 14: USART1 clock enable */
```

The code to configure the clocks follow the steps I have described
before. The conditional compilation allows the generation of the four
possible cases: HSI, HSE, PLL HSI and PLL HSE.

```c
/* By default SYSCLK == HSI [8MHZ] */

#ifdef HSE
/* Start HSE clock (8 MHz external oscillator) */
    RCC_CR |= RCC_CR_HSEON ;
/* Wait for oscillator to stabilize */
    do {} while( (RCC_CR & RCC_CR_HSERDY) == 0) ;
#endif

#ifdef PLL
/* Setup PLL HSx/2 * 6 [24MHz] */
    /* Default 0: PLL HSI/2 src, PLL MULL * 2 */
# ifdef HSE
    RCC_CFGR = RCC_CFGR_PLLSRC_HSE | RCC_CFGR_PLLXTPRE_DIV2 ;
# endif
    RCC_CFGR |= RCC_CFGR_PLLMUL( PLL) ;
    RCC_CR |= RCC_CR_PLLON ;
    do {} while( (RCC_CR & RCC_CR_PLLRDY) == 0) ;   /* Wait for PLL */

/* Switch to PLL as system clock SYSCLK == PLL [24MHz] */
    RCC_CFGR = (RCC_CFGR & ~RCC_CFGR_SW_MSK) | RCC_CFGR_SW_PLL ;
    do {} while( (RCC_CFGR & RCC_CFGR_SWS_MSK) != RCC_CFGR_SWS_PLL) ;
#else
# ifdef HSE
/* Switch to HSE as system clock SYSCLK == HSE [8MHz] */
    RCC_CFGR = (RCC_CFGR & ~RCC_CFGR_SW_MSK) | RCC_CFGR_SW_HSE ;
    do {} while( (RCC_CFGR & RCC_CFGR_SWS_MSK) != RCC_CFGR_SWS_HSE) ;
# endif
#endif

#ifdef HSE
/* Switch off HSI */
    RCC_CR &= ~RCC_CR_HSION ;
#endif
```

Systick reload value is calculated based on `CLOCK` constant value.

```
    SYSTICK_RVR = CLOCK / 8 - 1 ;   /* HBA / 8 */
```

Similarly, USART1 baud rate register is calculated based on `CLOCK` and
`BAUD` constant value.

```
    USART1[ BRR] = CLOCK / BAUD ;       /* PCLK is default source */
```

I add a debug print at the end of `init()` to display which clock
configuration has been set.

```c
    kputs(
#ifdef PLL
        "PLL"
#endif
#ifdef HSE
        "HSE"
#else
        "HSI"
#endif
        "\n") ;
```

## Build and test

To build, I first update the composition in Makefile.

`SRCS = startup.c clocks.c uptime.c`

Build complete successfully, this is for PLL HSE board configuration.

```
$ make
f030f4.elf
   text    data     bss     dec     hex filename
   1896       0      12    1908     774 f030f4.elf
f030f4.hex
f030f4.bin
```

I use a board with a 8 MHz quartz soldered on and test the four clock
configuration.

## Checkpoint

I have tuned the baud rate setting by using a higher frequency for the
system clock. The clock tree is complex and I have only look at a part
of it. Nevertheless the implementation for the clock configuration give
me some flexibility and ease of setup.

[Next]( 29_interrupt) I will implement interrupt driven transmission.

___
© 2020-2021 Renaud Fivet
