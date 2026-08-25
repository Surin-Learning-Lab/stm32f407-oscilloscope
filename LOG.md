# Engineering log

## 2026-08-23
Repo created. Nothing built yet.

The board has no dedicated SWD header, only the 20-pin JTAG/SWD connector (P1), and it isn't silkscreened. From the side furthest from the SD card mount, the bottom row is GND except pin 1(3.3v). pin4 is SWDIO and pin5 is SWCLK, use any pin on the bottom for GND except for pin1.

## 2026-08-24
the ARM GCC toolchain bundle isn't installed by the extension pack, the Bundles Manager UI failed with "no data provider registered," and installing via cube bundle install gnu-tools-for-stm32@14.3.1+st.2 plus a window reload fixed both that and the missing Devices and Boards panel.

the missing GCC bundle, the "no data provider registered" failure, the reload that fixed it, and the fact that the STM32Cube GDB launch option was wrong and the ST-Link one was right.

## 2026-08-24 — ADC1 first light (polled), and two debugging traps

Goal: prove ADC1 converts correctly on PA0 before adding timers or DMA.

### What happened

Configured ADC1 with IN0 on PA0, 480-cycle sampling time, polled conversion in
the main loop. First run read `raw = 0` for all three test conditions (PA0 to
GND, PA0 to 3V3, PA0 floating). A floating pin should read as noise, so a
constant zero meant the reading was not real.

Ruled out the obvious causes from the Debug Console rather than by guessing:

- `s1`, `s2` (HAL return codes) — both `HAL_OK`, so the ADC started and the
  conversion completed without error or timeout.
- `hadc1.Instance->SQR3` — `0`, so channel 0 was correctly selected.
- `GPIOA->MODER` — `0xA8000003`. Low two bits `11` = analogue mode. PA0
  correctly configured.
- `hadc1.Instance->DR` — **4095** with the 3V3 jumper in place.

DR held 4095 while `raw` held 0. The hardware was converting correctly the
whole time. The bug was in my code.

### Cause 1 — variable shadowing

I had declared `raw` twice:

    uint32_t raw = 0;              // before the loop — the one I was inspecting
    while (1) {
        uint32_t raw = HAL_ADC_GetValue(&hadc1);   // a SECOND, separate variable
    }

The inner `uint32_t` creates a new variable scoped to the loop body, which
shadows the outer one. The ADC value went into the inner variable; the outer
one stayed 0 forever. Compiles clean, no warning at default settings.

Fix: drop the type from the assignment so it writes to the existing variable.

    raw = HAL_ADC_GetValue(&hadc1);

### Cause 2 — optimised-away locals

Separately, locals that are assigned but never read get eliminated by the
optimiser. They have no storage, so the debugger's VARIABLES panel cannot show
them — they simply do not appear in the list. This is what happened to `s1` and
`s2` initially, and it looked like the build had not flashed.

Fix: declare debug-only variables `volatile` to force real memory storage.

    volatile uint32_t raw = 0;

### Results after the fix

| PA0 condition | raw   |
|---------------|-------|
| GND           | 0     |
| 3V3           | 4095  |
| Floating      | 804   |

Full scale at 3V3, zero at ground, arbitrary value when floating — correct
behaviour. 12-bit ADC, 0–4095 mapping to 0–3.3 V.

### Lessons

- A variable reading zero while the peripheral register reads correctly means
  the bug is in software, not hardware. Check the register directly and early.
- Missing entries in the VARIABLES panel usually means the optimiser removed
  them, not that the flash failed.
- The Debug Console accepts bare expressions (`hadc1.Instance->DR`), which is
  faster than rebuilding to add inspection variables.
- Declarations must sit inside `USER CODE` markers or CubeMX destroys them on
  the next regeneration.

  ## 2026-08-25 — Timer-triggered ADC, and why polling is a dead end

TIM2 configured for TRGO on update event, ADC1 external trigger set to
Timer 2 Trigger Out, rising edge. Polled from the main loop.

At 100 kHz trigger rate this fails immediately: HAL_ADC_PollForConversion
returns HAL_ERROR. Cause is ADC overrun. With EOCSelection at
ADC_EOC_SINGLE_CONV, overrun detection is enabled and every conversion must
be read. A main loop cannot read at 100 kHz, so OVR latches and the ADC
stops delivering data.

Slowing TIM2 to 10 Hz (PSC 8399, ARR 999) and raising the poll timeout to
200 ms proved the trigger chain works: ps = HAL_OK, raw = 4093 at 3V3,
1 at GND. Note a real 12-bit ADC does not hit the rails exactly — one or
two counts off is normal.

But even at 10 Hz it fails after the first breakpoint stop. TIM2 keeps
counting while the core is halted, so conversions pile up unread during
debugging and OVR latches again. Any breakpoint guarantees an overrun.

Conclusion: hardware-triggered conversion and software polling are
incompatible by construction. This is the concrete reason for DMA — it
services the data register at conversion rate with no CPU involvement, so
the core can be halted without losing data.

Also fixed this session: PA6 was assigned as the step-2 LED and conflicts
with ADC1_IN6. CubeMX flagged it with a warning triangle on ADC1. Reset the
pin to free it.

## 2026-08-25 — ADC to memory via DMA

Replaced polling with DMA (Direct Memory Access). ADC1 DMA request added in
CubeMX: circular mode, half-word data width on both peripheral and memory
sides, memory address increment on, peripheral increment off. ADC parameter
"DMA Continuous Requests" set to Enabled — circular mode on the DMA side
alone is not sufficient, the ADC must keep issuing requests.

Buffer: uint16_t adc_buf[1024] in main SRAM. Note DMA cannot access CCMRAM
(Core Coupled Memory) on this part, so sample buffers must live in the
128 KB main SRAM.

Start order matters: HAL_ADC_Start_DMA before HAL_TIM_Base_Start, so the
destination is armed before any conversion can complete.

TIM2 back to 100 kHz (PSC 0, ARR 839). Main loop now empty.

Verified by halting the core with the pause button and reading buffer slots
directly: 4095/4094/4094 with PA0 at 3V3, 1/2/1 at GND. Acquisition
continues while the core is halted, which is exactly what polling could not
do.

Not yet verified: the actual sample rate. A DC input looks identical at any
rate. Confirming the timebase needs a known-frequency signal source.