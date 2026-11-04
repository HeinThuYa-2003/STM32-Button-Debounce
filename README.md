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

### 5. Fix #3: state-based debounce in a timer interrupt (final version)
- **TIM2** generates an interrupt every **5 ms** (PSC = 83, ARR = 4999 at an 84 MHz timer clock)
- The ISR samples PA0 and counts how many samples in a row are identical
- A new state is accepted after **6 identical samples (6 × 5 ms = 30 ms)**
- The timer acts as the clock, so `HAL_GetTick()` is no longer needed
- The ISR only sets a `volatile` flag; UART printing stays in the main loop
- **Result:** reliable counting no matter how busy `while(1)` is. This is the pattern commonly used in real products.

| Method | Where | Release bounce | Affected by a busy main loop |
|---|---|---|---|
| Edge-based | EXTI callback | ❌ | No |
| State-based (polling) | `while(1)` | ✅ | Yes |
| **State-based (timer ISR)** | **5 ms TIM2 ISR** | ✅ | **No** |

## Key concepts
- EXTI / NVIC / vector table and `__weak` callbacks
- Flag pattern (keep ISRs short, no UART inside ISRs)
- `volatile` for variables shared between an ISR and main
- Why `HAL_Delay()` must not be used inside an ISR (it depends on SysTick)
- `strlen()` vs `sizeof()` for UART transmit length
- Edge-based vs state-based debounce
- Polling vs interrupts trade-offs
- Timer configuration: prescaler (PSC), auto-reload (ARR) and the period formula `T = (PSC+1)(ARR+1) / f_timer`
- 16-bit vs 32-bit timer limits
- Saturating counters (defensive programming against `uint8_t` overflow)

## How to build
1. Open the project in STM32CubeIDE
2. Open the `.ioc` file and click **Generate Code** (this regenerates the `Drivers/` folder)
3. Build and flash to a Nucleo-F401RE
4. Open a serial terminal at **115200 baud**

## Next steps
- Add low-power sleep with `__WFI()` between timer interrupts
- Support multiple buttons using arrays
- Add PWM output (e.g. LED brightness controlled by button presses)
