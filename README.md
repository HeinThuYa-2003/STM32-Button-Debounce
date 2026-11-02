# STM32 Button Debounce: Measuring and Fixing Switch Bounce

Firmware for the **STM32 Nucleo-F401RE** that counts button presses and logs them over UART. I used it to measure button bounce and then fix it, first with an edge-based method and then with a state-based one.

## Hardware
- STM32 Nucleo-F401RE
- On-board user button **B1** (PC13)
- A jumper wire on **PA0 (A0)** touched to **GND**, used as a deliberately "bad" switch

## Tools
- STM32CubeIDE / CubeMX (HAL)
- Serial terminal at 115200 baud (USART2 via ST-Link Virtual COM Port)

## What it does

### 1. B1 counter: EXTI + flag pattern
- PC13 is configured as an EXTI falling-edge interrupt
- The ISR only increments a counter and sets a `volatile` flag
- The main loop checks the flag and does the slow work (`sprintf` + `HAL_UART_Transmit`), keeping the ISR short

### 2. Measuring bounce
- PA0 is configured as an EXTI input with an internal pull-up
- One touch of the jumper wire to GND produced **26 interrupts** (count jumped from 15 to 41)
- The on-board B1 showed no bounce because of the RC filter on the Nucleo board (hardware debounce)

### 3. Fix #1: edge-based debounce (`HAL_GetTick`)
- An edge is ignored if it arrives less than 200 ms after the last accepted press
- **Limitation found:** release bounce was still counted as a new press when the wire was held for more than 200 ms

### 4. Fix #2: state-based debounce (polling)
- PA0 is read continuously in the main loop
- A new state is accepted only after the pin has been stable for **30 ms**
- Presses are counted only on a stable HIGH → LOW transition
- **Result:** 10 touches give exactly 10 counts, and releases are not counted 

## Key concepts
- EXTI / NVIC / vector table and `__weak` callbacks
- Flag pattern (keep ISRs short, no UART inside ISRs)
- `volatile` for variables shared between an ISR and main
- Why `HAL_Delay()` must not be used inside an ISR (it depends on SysTick)
- `strlen()` vs `sizeof()` for UART transmit length
- Edge-based vs state-based debounce
- Polling vs interrupts trade-offs

## How to build
1. Open the project in STM32CubeIDE
2. Open the `.ioc` file and click **Generate Code** (this regenerates the `Drivers/` folder)
3. Build and flash to a Nucleo-F401RE
4. Open a serial terminal at **115200 baud**

## Next steps
- Move the state-based debounce into a periodic **timer interrupt** (the industry pattern)
- Add low-power sleep with `__WFI()`
