# TCD1304 Linear CCD Interface using STM32F401CCU6

A firmware project for interfacing the STM32F401CCU6 microcontroller with the TCD1304 linear CCD sensor using hardware timers, ADC + DMA acquisition, and USB streaming.

This project generates all required CCD timing signals entirely in hardware:

- fM (master clock)
- SH (sample & hold)
- ICG (integration clear gate)
- ADC sampling trigger

The STM32 continuously captures CCD pixel data and streams frames to a host PC over USB CDC.

---

# Relevant Articles and Video Series

A complete video playlist covering the development of relevant projects is available here:

https://www.youtube.com/playlist?list=PLaeIi4Gbl1T_VSpqzWGMiYx2A7urjnoD4

Please make sure to check the descriptions of the videos in the playlist, as each video is accompanied by a detailed article relevant to the topic being discussed.

---

# Support My Work

If you use my source code, articles, or project ideas, please consider supporting my work. Developing these projects requires significant time, testing, and equipment costs.

## Donation

PayPal donation link:

https://www.paypal.com/donate/?hosted_button_id=8JWLLFLE2TLCA

## YouTube Channel Membership

Support the channel directly by joining the membership program:

https://www.youtube.com/channel/UCKp1MzuAceJnDqAvsZl_46g/join

## Affiliate Links

Support future projects by using the affiliate links listed here:

https://curiousscientist.tech/tools

---

# Citation Notice

If you reuse this code, documentation, or project structure in your own work, please make sure to properly cite and reference the original source.

# TCD1304 Linear CCD Interface using STM32F401CCU6

## Overview

This project interfaces a **TCD1304 linear CCD sensor** with an **STM32F401CCU6** microcontroller using:

* Hardware timers for CCD timing generation
* ADC + DMA for high-speed pixel acquisition
* USB CDC for frame transmission to a PC
* Adjustable integration time control

The firmware generates all required CCD timing signals:

* `fM` (master clock)
* `SH` (sample & hold)
* `ICG` (integration clear gate)
* ADC sampling trigger

The STM32 continuously captures CCD pixel data and streams frames to a host PC over USB.

---

# Hardware Architecture

## MCU

* MCU: STM32F401CCU6
* Core: ARM Cortex-M4
* Clock configuration:

  * External HSE crystal enabled
  * PLL configured to 84 MHz system clock

---

# CCD Timing Architecture

The TCD1304 requires precise timing relationships between:

1. Master clock (`fM`)
2. Integration control (`SH`)
3. Readout reset (`ICG`)
4. ADC sampling clock

This project uses multiple synchronized STM32 timers to generate these signals entirely in hardware.

---

# Timer Assignments

| Function          | Timer | Channel | STM32 Pin | Purpose                        |
| ----------------- | ----- | ------- | ---------- | ------------------------------ |
| ICG               | TIM2  | CH1     | PA5        | Master synchronization trigger |
| fM / MCLK         | TIM3  | CH1     | PA6        | CCD master clock               |
| ADC trigger clock | TIM4  | CH4     | PB9        | ADC synchronization            |
| SH pulse          | TIM5  | CH4     | PA3        | Integration timing             |
| CCD analog output | ADC1  | IN1     | PA1        | CCD analog video input         |

---

# Timer Working Principles

## TIM2 — ICG Generator (Master Timer)

TIM2 acts as the master synchronization timer for the CCD acquisition sequence.

### Configuration

* PWM mode enabled
* TRGO output enabled
* Master/slave synchronization enabled
* PWM polarity: LOW

### Purpose

The ICG pulse:

* Resets the CCD shift register
* Defines the frame period
* Synchronizes the SH timing
* Starts the acquisition cycle

### Key Parameters

| Parameter | Value     |
| --------- | --------- |
| Prescaler | 0         |
| Period    | 1241184-1 |
| Pulse     | 840-1     |

---

## TIM3 — fM (Master Clock)

TIM3 generates the CCD master clock (`fM`).

### Purpose

The master clock shifts pixel charges through the CCD internal register.

### Configuration

* PWM mode
* 50% duty cycle
* High-speed continuous clock generation

### Key Parameters

| Parameter | Value |
| --------- | ----- |
| Prescaler | 0     |
| Period    | 84-1  |
| Pulse     | 42-1  |

This produces a 1 MHz square wave with a 50% duty cycle when running from the 84 MHz timer clock.

---

## TIM4 — ADC Sampling Clock

TIM4 generates the ADC timing clock.

### Purpose

This timer controls when ADC conversions occur relative to CCD pixel output timing.

The ADC sampling timing must be synchronized with:

* CCD master clock
* CCD analog output settling
* Pixel transition timing

### Operation

TIM4 PWM output is started before the CCD acquisition begins.

The ADC uses DMA to continuously transfer converted pixel values into memory.

### Note

The clock signal is available on PB9 pin, but the signal is only used internally for triggering the ADS conversions.

The signal however is very useful for troubleshooting and understanding the timers and timings. 

---

## TIM5 — SH (Sample & Hold)

TIM5 generates the SH pulse.

### Purpose

The SH pulse controls:

* Exposure time
* Integration duration
* Pixel charge accumulation period

### Configuration

TIM5 operates in slave-trigger mode and is synchronized from TIM2.

### Key Parameters

| Parameter | Value   |
| --------- | ------- |
| Prescaler | 0       |
| Period    | 29552-1 |
| Pulse     | 378-1   |

---

## Predefined SH / Integration Values

The following predefined values can be used for the TIM5 auto-reload register (`ARR`) to control the CCD integration time:
```text
1241184
0620592
0413728
0310296
0206864
0177312
0155148
0103432
0088656
0077574
0059104
0051716
0044328
0038787
0029552
0025858
0022164
0014776
0012929
0011082
0007388
0005541
0003694
0001847
```

The lowest value 1847 is 21.98 us (21979 ns) integration time, and the highest value 1241184 is 14.77 ms (14770089 ns) integration time.
The values are carefully adjusted for the master clock (fM) and the ICG period. If you use different values for them, you must readjust the SH values as well so the strict timing is maintained!

---

# Adjustable Integration Time

Integration time is dynamically adjustable over USB.

The firmware receives a command:

```text
iXXXXXXX
```

Where:

* `i` = integration command
* `XXXXXXX` = 7-digit timer period value

Example:

```text
i0012345
```

The firmware then updates:

```c
__HAL_TIM_SET_AUTORELOAD(&htim5, SH_value-1);
```

This changes the SH timer period and therefore modifies the CCD exposure time.

---

# ADC + DMA Acquisition

## ADC Operation

The CCD analog output is connected to ADC1.

ADC conversions occur continuously while the timing signals are active.

---

## DMA Operation

DMA transfers ADC samples directly into RAM without CPU intervention.

### Buffer

```c
volatile uint16_t CCDDataBuffer[CCDSize];
```

This buffer stores one complete CCD frame.

### Advantages

Using DMA:

* Minimizes CPU load
* Prevents missed samples
* Allows high-speed acquisition
* Enables continuous streaming

---

# Frame Transfer over USB

## USB CDC

The STM32 appears as a USB virtual COM port.

Captured frames are transmitted using:

```c
CDC_Transmit_FS((uint8_t*)CCDDataBuffer, 2*CCDSize);
```

Each pixel is transmitted as:

* 16-bit unsigned integer
* Binary data format

---

# Acquisition Sequence

## Start Command

When the PC sends:

```text
s
```

The firmware starts:

1. ADC DMA acquisition
2. ADC timing PWM (TIM4)
3. CCD master clock (TIM3)
4. SH timing (TIM5)
5. ICG trigger generation (TIM2)

---

## Stop Command

When the PC sends:

```text
d
```

The firmware:

1. Stops ADC DMA
2. Stops all timers
3. Clears timer counters
4. Clears timer update flags

This guarantees clean restart behavior.

---

# Firmware Data Flow

```text
TCD1304 CCD
    |
    | Analog Video Output
    v
ADC1
    |
    v
DMA2
    |
    v
CCDDataBuffer[]
    |
    v
USB CDC
    |
    v
PC Application
```

---

# GPIO Usage

## Configured GPIOs

| Pin  | Function    |
| ---- | ----------- |
| PC13 | Status LED  |
| PA0  | GPIO output |
| PA2  | GPIO output |
| PA4  | GPIO output |

---

# Software Structure

## Main Loop

The firmware main loop continuously checks USB commands:

```c
while (1)
{
    checkUSBCommunication();
}
```

---

# USB Command Interface

| Command    | Description          |
| ---------- | -------------------- |
| `s`        | Start acquisition    |
| `d`        | Stop acquisition     |
| `iXXXXXXX` | Set integration time |

---

# Synchronization Strategy

The firmware heavily relies on STM32 timer synchronization features.

Key design principles:

* Hardware-generated timing
* Minimal CPU involvement
* DMA-based acquisition
* Deterministic CCD timing
* USB frame streaming

This architecture ensures stable CCD operation and reliable high-speed image acquisition.

---

# Project Summary

This firmware implements a complete hardware-timed acquisition system for the TCD1304 line scan CCD using the STM32F401CCU6.

The design uses:

* Multiple synchronized hardware timers
* DMA-driven ADC acquisition
* USB streaming
* Adjustable exposure control

The result is a compact, high-speed line scan camera platform suitable for:

* Spectroscopy
* Optical sensing
* Industrial inspection
* Scientific imaging
* DIY spectrometers
* Embedded vision systems
