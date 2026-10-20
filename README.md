# GFXTIM Peripheral Driver

This module provides an abstraction layer for the **graphic timer** (`GFXTIM`) of STM32 MCUs: the timing infrastructure of a graphical
application that is synchronized to the display - the **frame and line clocks** derived from the display synchronization or from a
free-running clock, the **absolute and the relative frame counters**, the line timer with the compare channels, the detection and the
synchronization to the **tearing-effect signal** of a smart panel, the **event generators** that trigger the graphics engines and
the watchdog counter. It is a low-level MCAL module: it owns the GFXTIM hardware, the application and the other modules never touch the registers.

The public interface (`GfxTim_Port.h`, `GfxTim_Types.h`) is **identical for all supported families**. The implementation
for a family lives in its own branch (`Dev/<Family>` for development, `Releases/<Family>` for the released versions), so
a module or an application written against the interface does not change when the MCU family changes.

This README is the landing page and the **specification of the interface** (master). The family branches contain their
own README with everything that is specific to the family: the instances of the device, their settings and the status of
the implementation.

> **Status: design.** No implementation exists yet, the interface below is a proposal. Stories and branches follow.

---

## Intent and purpose

- A graphics application lives in **frames**: draw the next frame while the current one is displayed, switch at the right moment, animate
  at a fixed rate, refresh a part of the screen when the panel's tearing-effect line says it is safe. The software for this is full of timers, interrupts
  and frame counters that drift against the display. The GFXTIM is a **timer whose time unit is the frame and the line** of the display - it counts the frames
  and the lines from the same synchronization the display uses - and generates the events at the programmed frame / line numbers.
- The typical uses are the **animation timeline** (an event every N frames, the relative frame counters reload automatically), the **frame time-stamp** (the absolute frame
  counter for the measurement of the rendering time and the dropped frames), the **start of the rendering at a line** of the frame (the line timer compare triggers the graphics
  engine so that the drawing stays ahead of the beam), the **tearing-effect synchronization** (the signal from a command-mode panel is detected and its timing is reported),
  and the **watchdog** that notices a stuck graphics pipeline (no frame event in time).
- The module describes these as *"every 2nd frame call this"*, *"trigger the DMA2D at the line 100 of every frame"*, *"alarm if no frame in 50 ms"* and hides the clock generator
  dividers, the compare registers and the event selection.
- The module hides the differences between the families: the clock sources, the event outputs and the number of the counters differ, the interface stays the same.

What the module is **not**: it does not draw or display (the Dma2D and the Ltdc modules do), and it does not generate the display synchronization (the LTDC does; the GFXTIM
listens to it).

---

## Concept

| Item | Meaning |
|---|---|
| **Clock generator** | The source of the frame and the line clock: the LTDC synchronization (`GFXTIM_CLK_LTDC`), an external synchronization pin, or a free-running system clock with the divider that gives a virtual frame rate and the line period. |
| **Absolute timer** | A frame counter (and a line counter) with a compare channel: the absolute time since the start, an event at a given frame. |
| **Relative timers** | Two auto-reload frame counters: periodic events every N frames, the one-shot mode. |
| **Line timer** | A line counter with two compare channels: the events at the programmed line of the frame. |
| **Tearing effect** | The input line of the panel, the edge and the source selection, the measurement of its timing. |
| **Event generators** | Four events (a frame start, a line compare, a relative timer reload ...) that can be routed as the **triggers** to the other peripherals (`GFXTIM_TRIGGER_DMA2D`, `_LTDC_RELOAD` ...). |
| **Watchdog** | A counter reloaded by the frame events; the event when it expires. |

Rules:

1. `GfxTim_Init` configures the clock generator and the counters from the configuration, enables the timer and leaves the events disabled; each event is armed with
   its own function (`GfxTim_Set_EventActive`).
2. The frame rate of a free-running clock is configured in Hz and the line period in the number of the lines per frame; the module calculates the dividers and returns the achieved
   values. The clock source is changed only with the timer stopped.
3. A compare event is reported to the callback with the counter value; periodic relative events are repeated by the hardware until disabled.
4. A trigger output is routed with `GfxTim_Set_TriggerOutput( event, target )`; the target must exist on the device and the connection is checked.
5. The tearing-effect detection reports the polarity-corrected edge and the measured period (the callback `TearingEffectCallback`).
6. All parameters are checked before a register is written.

---

## Supported families

Every family gets its own story, its own tasks and its own branches `Dev/<Family>` and `Releases/<Family>` cut from this
master.

| Family | Branch | Timer | Status |
|---|---|---|---|
| STM32H5 | `Dev/STM32H5` | `GFXTIM` on the STM32H5E / H5F lines: frame and line clock generator, one absolute frame counter with a compare channel, two auto-reload relative frame counters, a line timer with two compare channels, the tearing-effect management, four event generators with the trigger outputs, one watchdog | planned (first) |
| STM32U5 | `Dev/STM32U5` | `GFXTIM` on the lines with the graphics | planned |
| STM32H7 | `Dev/STM32H7` | device dependent | planned |
| STM32U3 | `Dev/STM32U3` | device dependent | planned |
| STM32F4, STM32F7, STM32G4, STM32L4 | `Dev/STM32F4` ... | no graphic timer on the device (the line and frame events of the `LTDC` and a general purpose timer replace it) - the module is a stub that returns `GFXTIM_REQUEST_ERROR` | planned |

A function without a meaning in a family stays in the interface, returns `GFXTIM_REQUEST_ERROR` and suppresses the unused
parameters (for example a trigger target that the device does not have).

---

## Public interface in short (proposal)

| Group | Functions |
|---|---|
| Module management | `GfxTim_Get_ModuleVersion`, `GfxTim_Init`, `GfxTim_Deinit`, `GfxTim_Task`, `GfxTim_Get_DefaultConfig` |
| Clock | `GfxTim_Set_ClockConfig`, `GfxTim_Get_FrameRate` |
| Timers | `GfxTim_Set_AbsoluteCompare`, `GfxTim_Get_AbsoluteFrame`, `GfxTim_Set_RelativeTimer`, `GfxTim_Set_LineCompare`, `GfxTim_Get_LinePosition` |
| Tearing effect | `GfxTim_Set_TearingEffectConfig`, `GfxTim_Get_TearingEffectPeriod` |
| Events and triggers | `GfxTim_Set_EventActive`, `GfxTim_Set_EventInactive`, `GfxTim_Set_TriggerOutput` |
| Watchdog | `GfxTim_Set_WatchdogConfig`, `GfxTim_Set_WatchdogActive`, `GfxTim_Set_WatchdogInactive` |
| Interrupts | `GfxTim_Set_IrqPriority` / `Get`, the callbacks are a part of the configuration |
| Pins | `GfxTim_InitTearingEffectGpio` |

```c
gfxTim_ModuleVersion_t GfxTim_Get_ModuleVersion     ( void );

gfxTim_RequestState_t  GfxTim_Init                  ( const gfxTim_Config_t * const gfxTimConfig );
gfxTim_RequestState_t  GfxTim_Deinit                ( void );
void                   GfxTim_Task                  ( void );
gfxTim_RequestState_t  GfxTim_Get_DefaultConfig     ( gfxTim_Config_t * const gfxTimConfig );

gfxTim_RequestState_t  GfxTim_Set_ClockConfig       ( const gfxTim_ClockConfig_t * const clockConfig );
gfxTim_RequestState_t  GfxTim_Get_AbsoluteFrame     ( gfxTim_Frame_t * const frame );

gfxTim_RequestState_t  GfxTim_Set_RelativeTimer     ( gfxTim_RelativeId_t timerId, gfxTim_Frame_t reload, gfxTim_Mode_t mode );
gfxTim_RequestState_t  GfxTim_Set_LineCompare       ( gfxTim_CompareId_t compareId, gfxTim_Line_t line );
gfxTim_RequestState_t  GfxTim_Set_TriggerOutput     ( gfxTim_Event_t event, gfxTim_Trigger_t trigger );
gfxTim_RequestState_t  GfxTim_Set_EventActive       ( gfxTim_Event_t event );
gfxTim_RequestState_t  GfxTim_Set_WatchdogConfig    ( gfxTim_Frame_t timeoutFrames );
```

Every function returns `GFXTIM_REQUEST_OK` / `GFXTIM_REQUEST_ERROR` and checks its parameters and the state of the module
before it touches a register.

| Type | Meaning |
|---|---|
| `gfxTim_Config_t` | The clock configuration, the timers, the events, the tearing-effect configuration, the watchdog, callbacks, interrupt priority |
| `gfxTim_Event_t` | `GFXTIM_EVENT_FRAME_START`, `_RELATIVE_1_RELOAD`, `_RELATIVE_2_RELOAD`, `_ABSOLUTE_COMPARE`, `_LINE_COMPARE_1`, `_LINE_COMPARE_2`, `_TEARING_EFFECT`, `_WATCHDOG` |
| `gfxTim_Trigger_t` | The trigger targets of the device (`GFXTIM_TRIGGER_DMA2D` ...) |
| Callbacks | `EventCallback( event, counterValue )`, `TearingEffectCallback`, `WatchdogCallback` |

---

## Usage

### Start the drawing at the line 20 of every frame and animate every second frame

```c
#include "GfxTim_Port.h"

gfxTim_Config_t gfxTimConfig;

(void)GfxTim_Get_DefaultConfig( &gfxTimConfig );
gfxTimConfig.Clock.Source = GFXTIM_CLK_LTDC;                  /* the frame is the real display frame */
(void)GfxTim_Init( &gfxTimConfig );

(void)GfxTim_Set_LineCompare( GFXTIM_COMPARE_1, 20u );
(void)GfxTim_Set_TriggerOutput( GFXTIM_EVENT_LINE_COMPARE_1, GFXTIM_TRIGGER_DMA2D );
(void)GfxTim_Set_EventActive( GFXTIM_EVENT_LINE_COMPARE_1 );

(void)GfxTim_Set_RelativeTimer( GFXTIM_RELATIVE_1, 2u, GFXTIM_MODE_CONTINUOUS );   /* every second frame */
(void)GfxTim_Set_EventActive( GFXTIM_EVENT_RELATIVE_1_RELOAD );                   /* EventCallback: advance the animation */
```

---

## Behaviour and limits

- **Clock source.** With the LTDC synchronization the counters follow the real display; with a free-running clock the frame rate is virtual (for a panel with its own refresh or for the
  tests).
- **Event latency.** The callbacks run in the interrupt; a drawing started from the callback competes with the other work - the trigger output (the hardware connection) is the
  deterministic way.
- **Watchdog.** The watchdog counts frames: a stuck display pipeline is noticed within the programmed number of frames, it does not reset the MCU by itself.
- **Not part of the first version:** the programming of the panel (the command mode refresh), the adaptive frame rate, the complete timeline engine on the top of the events.

---

## Dependencies

- **Rcc** - peripheral clock and reset
- **Nvic** - interrupt
- **Gpio** - the tearing-effect input pin
- **Ltdc** - the display synchronization that the timer follows (configured by its module)
- **RAL (Register Abstraction Layer)** - used internally to access the registers

All mandatory RCC, NVIC and GPIO configurations are handled internally.

---

## Testing

- **Unit tests** (host, Unity + CMock + RegMem): the clock divider calculation, the compare and the reload register mapping, the event and the trigger routing, the tearing-effect
  period measurement, the watchdog, the callbacks, the rejected configurations.
- **Integration tests** (target, probe-rs): the frame events against the LTDC refresh rate measured by a timer, the line compare trigger of a graphics transfer, the watchdog with a
  stopped display.

---

## Interface conventions

The module follows the common rules of the MCAL layer, so that it behaves like the other modules:

- **Lists instead of free members.** Everything that the hardware restricts (the pins, the kernel clock sources, the DMA
  channels, the triggers and the other selectable sources) is a `typedef enum` list with exactly the items that are valid
  for the device line; a pin is an item `GFXTIM_<SIGNAL>_PIN_<INSTANCE>_P<port><pin>` and there is an `*_UNUSED` item. A choice
  that is not in the list cannot be written.
- **Own peripheral only.** The module calls only the registers / LL functions of its own peripheral. The clocks, the
  interrupts, the pins, the DMA and the EXTI lines go through the API of the Rcc, Nvic, Gpio, Gpdma and Exti modules; a
  missing function is added to that module, it is not bypassed.
- **Init verifies itself.** `GfxTim_Init` checks the result at the end; when it is not as required, the module is
  deinitialized and an error is returned. The waiting for the hardware is a loop on the register state with a timeout,
  never a blind delay.
- **Transfer modes.** Where the module moves data, the POLL (served by `GfxTim_Task`), ISR and DMA modes have the same
  configuration structure, the same request and the same callbacks (error, half transfer, transfer complete).
- **Flags and states are enums of the module** (`gfxTim_FunctionState_t`, `gfxTim_FlagState_t`), the limits and the constants are
  named, there are no magic numbers in the interface.
- **Unsupported functions stay.** A function without a meaning in a family keeps its signature, suppresses the unused
  parameters and returns `GFXTIM_REQUEST_ERROR`.
- **Documentation.** Every function and type has a Doxygen description, the main header has the `\defgroup`.

---

## Repository layout and branches

The master holds this README and the license. The family branches (`Dev/<Family>`, `Releases/<Family>`) hold:

| Path | Content |
|---|---|
| `README.md` | README of the family: the instances of the device, specifics, errata, status |
| `GfxTim_Port.h`, `GfxTim_Types.h` | Public interface (identical in all families) |
| `GfxTim*.c`, `GfxTim.h` | Implementation of the family |
| `CMakeLists.txt` | Library `GfxTim_Lib` of the EmBi platform |
| `Tests/UnitTests` | Unit tests (Unity + CMock + RegMem) |
| `Tests/IntegrationTests` | Integration tests on the target (probe-rs) |

A change of the public interface is made in all families.

---

## CMake integration

```cmake
target_link_libraries( User_Lib PRIVATE GfxTim_Lib )
```

1. Link `GfxTim_Lib` to your CMake library.
2. Include `GfxTim_Port.h` in your project.
3. Configure the module as needed for your hardware.

---

## License

This project is licensed under the **Creative Commons Attribution-NonCommercial 4.0 International (CC BY-NC 4.0)**.

You are free to use, modify, and share this work for **non-commercial purposes**, provided appropriate credit is given.

See [LICENSE.md](LICENSE.md) for full terms or visit [creativecommons.org/licenses/by-nc/4.0](https://creativecommons.org/licenses/by-nc/4.0/).

---

## Authors

- **Mr.Nobody** - [embedbits.com](https://embedbits.com)

Contributions are welcome! Please open a pull request.
