# 3.2: DHT11 Errata

I only did basic testing so far, checking that the value read were
displayed properly. But the humidity values didn’t seem correct, so I
need to do some extra verifications. There are many possible causes and
they can combine, bugs like humans are social animals, when you find
one, look for its mate.

Some candidates for investigations:

- Coding or wiring mistakes.

- Wrong interpretation of the datasheet.

- Mistake in the datasheet.

- Bad sample of the DHT11.

- Deviation from the specifications.

- Sensitivity to power supply, either the accuracy of the voltage or the
  fact I am using 3.3V instead of 5.5V.

On top of this I need to do extra testing

- Test at temperature below 0°C.

- Check if the readings are stable or jittery.

## Coding and wiring check

My basic test shows that I manage to read values without errors. A loose
wire would generate Timeout errors or Checksum errors. The temperature
values look reasonable, only humidity seems way too high.

I double checked both code and wiring to be on the safe side.

I am confident that the protocol implementation matches the transmission
of DHT11. So next I need recheck if the setup time (one second) and
frequency of reading (every two seconds) are the correct requirement.

## Datasheet check

I rechecked Aosong website for the latest version of the DHT11
datasheet. There is only a Chinese version v1.3 branded ASAIR. I have
the previous version v1.3 branded AOSONG which seems identical except a
revision page.

I also have an English Aosong DHT11 datasheet that seems outdated.
Aosong doesn’t seem to allow redistribution of their datasheet so most
online vendor have made their own based on that English version.

The English datasheet states the temperature range as 0~50℃, the
humidity range to be 20~95%RH and recommend a reading frequency greater
than 5 seconds.

The Chinese datasheet states the temperature range as -20~60°C, the
humidity range to be 0~95%RH and recommend a reading frequency greater
than 2 seconds. It also explains the encoding of negative temperature.

My implementation is based on the latest Chinese version. I can retest
with a longer interval between readings in case I am using a chip that
follows the older specification (5 seconds interval instead of 2).

In **dht11main.c** I just need to change the test `if( 1 & last)` to
`if( 2 == (last % 15))` in order to read every 15 seconds after a setup
time of 2 seconds. If this works better, I can retry for shorter
intervals by changing the modulo value.

When I test with readings every 15 seconds, I get stable humidity value
around 25~26%RH. As I have changed both interval and setup time, I need
confirm that it is the interval time that matters.

With intervals of 5 and 6 seconds, the reading jumps above 37%RH. So
it’s clearly a problem with the interval.

I want to make a round number of samples per minute, so I need retest to
check if 10 and 12 seconds work the same as 15, but before I do fine
tuning, I better check if this is not just a problem with that
particular DHT11.

## Product quality

It’s clear that the DHT11 I am testing is not behaving according to old
or new specifications for the sampling interval.

Defects happen! No production line output is 100% perfect, so I may just
have a defective or damaged chip. As this particular product is low
cost, I have several on hands I bought from different suppliers, I would
be very unlucky if they end up all coming from the same bad production
batch or all damaged by mishandling.

Testing the four DHT11 I have, I find that three of them are working in
their precision range when sampled every five seconds. Understanding
that humidity precision is ±5%RH and temperature precision is ±2℃, there
is still room for quite some variation between readings from different
devices.

I select the most accurate chip at my current environment humidity and
temperature to do further test.

## Voltage tolerance

When it come to measurement, precision is often related to the value of
the reference voltage. I want to check the difference in measurement of
the same chip when powered at 5V compared to 3.3V.

I need to use a 5V tolerant GPIO pin for this test, so I switch to GPIOA
13 (SWDIO). By default that pin is configured as ALT SWDIO, floating
input with weak pull-up, similar to the initial state for DHT11 Data IO
pin.

The board needs to be powered by its USB connector 5V source instead of
directly by the USB to Serial adapter, I make sure board and adapter are
powered independently being connected only by TX, RX and GND. I can then
select the voltage of DHT11 according to what I want to test, 3.3V or
5V.

There is a difference in measurement, 3.3V giving slightly higher value
than 5V, for this particular test: 2%RH more for humidity and 0.3℃ for
temperature. There is no clear advantage to use 5V over 3.3V.

I am not doing precise voltage test as the precision of DHT11 and the
variation between the chips I have would make the interpretation of the
results irrelevant.

## Temperature below 0℃.

The Chinese version of the datasheet gives the encoding for temperature
below zero ℃ and a measurement range of -20~60℃.

I have implemented `dht11_read()` accordingly so I just need to test at
below zero ℃.

From my test, I can see that the values reported are negative but I
found a difference versus the datasheet.

According to the datasheet, the temperature values are encoded as 1 byte
for the integer part and 1 byte for the one decimal digit fractional
part, the highest bit of the fractional part indicating the sign.

So when temperature crosses zero, I expects to see

```
0 + 2 => 0.2
0 + 1 => 0.1
0 + 0 => 0.0
0 - 1 => -0.1
0 - 2 => -0.2
...
0 - 8 => -0.8
0 - 9 => -0.9
1 - 0 => -1.0
1 - 1 => -1.1
...
```

Instead the values transmitted are

```
0 + 2 => 0.2
0 + 1 => 0.1
0 + 0 => 0.0
0 - 9 => ???
0 - 8 => ???
...
0 - 2
0 - 1
0 - 0 => !!!
1 - 9
...
```

I have to modify my original implementation

```
    dht11_tempc = values[ 2] ;
    dht11_tempf = values[ 3] ;
    if( dht11_tempf & 0x80) {
        dht11_tempc *= -1 ;
        dht11_tempf &= 0x7F ;
    }
```

And retest after the following modification.

```
    dht11_tempc = values[ 2] ;
    dht11_tempf = values[ 3] ;
    if( dht11_tempf & 0x80) {
        dht11_tempc *= -1 ;
        dht11_tempf = 10 - ( dht11_tempf & 0x7F) ;
        if( dht11_tempf == 10) {
            dht11_tempc -= 1 ;
            dht11_tempf = 0 ;
        }
    }
```

## Stability

During my test I didn’t noticed big surges in measurements but the time
to get to actual value is quite long. The interval between readings
affects the measurement and initially it takes a long time for the
readings to converge to actual temperature or humidity.

## Conclusions

I didn’t find any English version of the latest version of the
datasheet.

I found some difference between the Chinese datasheet and the behavior
of the chips I have when it come to representation of temperature below
0℃.

Cost and one pin transmission interface are the two main advantages of
this chipset.

I could use the DHT11 for monitoring temperature and humidity
variations. For fast and accurate measurements, this is not the droid I
am looking for.

## Checkpoint

I did multiple checks and found several issues. I can secure stable
readings by controlling the reading interval, which means I can tune the
timing to suit a specific chip. There is variations in readings between
chips and due to the loose precision it is hard to understand how good
or how bad the measurements are.

[Next]( https://warehouse.motd.org/?page_id=908) I will use another
digital thermometer as a reference.

___
© 2020-2021 Renaud Fivet
