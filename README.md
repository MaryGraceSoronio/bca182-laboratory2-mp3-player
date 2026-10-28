# Personal MP3 Player

A compact embedded music player built for the RT-Thread RT-Spark development board featuring an STM32F407 microcontroller. The project uses PlatformIO, STM32Cube HAL, and FreeRTOS to implement a real-time, multitask audio player with a small LCD display, tactile controls, LEDs, and UART feedback.

- Course: BCA182 Embedded Systems Programming
- Institution: Mindanao State University - Iligan Institute of Technology
- Authors: Sherie Grace L. Lupoz, Renato G. Beroy, Binyusoph P. Sansarona
- Hardware: RT-Spark (STM32F407ZGT6, 168 MHz, 8 MHz crystal)
- Platform: STM32 using PlatformIO + STM32Cube + FreeRTOS

## Overview

This project simulates a portable MP3-style player on embedded hardware. It plays a set of pre-defined songs using an ES8388 audio codec, updates the LCD with playback status and song information, responds to user inputs using onboard keys and a user button, and changes playback tempo and volume through hardware controls.

Unlike a full software MP3 decoder, this is a tone-synthesis-based embedded player. The firmware converts note periods into audio waveform data, streams them over I2S, and manages timing with FreeRTOS software timers.

## Features

- Real-time audio playback through ES8388 codec over I2S3
- Song selection using binary key entry on three selection buttons
- 5-second confirmation window before changing tracks
- Control of play/pause behavior using USER_BUTTON
- Tempo adjustment by holding the USER_BUTTON
- Analog volume control using a potentiometer and ADC input
- LCD status display for current song, playback mode, and volume
- UART startup instructions and song list display
- LED status indicators for playback, paused/stopped, and selection mode
- Multitask design using FreeRTOS threads and timers

## Hardware

### Main board and processor

- RT-Spark board with STM32F407ZGT6 MCU
- Cortex-M4F running at 168 MHz
- External 8 MHz crystal
- ST-LINK programmer/debugger interface

### Audio hardware

- ES8388 audio codec
- I2S3 data stream for digital audio output
- Headphone jack output on the board
- Codec controlled over I2C2

### User interface hardware

- 4 onboard keys: UP, LEFT, DOWN, RIGHT
- USER_BUTTON on breadboard connected to PA0 and GND
- 240x240 LCD (ST7789 via FSMC)
- Three LEDs for status indication
- Potentiometer for volume adjustment

## Control scheme

Buttons 1-4 are the onboard keys (UP, LEFT, DOWN, RIGHT). The USER_BUTTON is a pushbutton mounted on the breadboard.

| Input | Action |
|---|---|
| Hold LEFT / DOWN / RIGHT | Enter song selection code in binary: LEFT = 4, DOWN = 2, RIGHT = 1 |
| Tap UP | Begin the song selection flow. Key order does not matter. |
| Tap UP again within 5 s | Confirm the selected song. Playback begins from the start. |
| Wait 5 s without confirmation | Cancel song selection and continue current playback. |
| USER_BUTTON tap | Toggle play / pause |
| USER_BUTTON hold 1.5 s | Increase playback tempo; wraps back to slowest value |
| Potentiometer | Adjust volume |

### LED meanings

| LED | Meaning |
|---|---|
| Yellow | Playing |
| Red | Paused or stopped |
| Green | Song selection waiting for confirmation |

## Song selection and supported songs

The binary selection inputs use the three code buttons. Each code is formed as:

- LEFT = 4
- DOWN = 2
- RIGHT = 1

The valid song codes are 000 through 111, but because only the first 8 songs are exposed by the handout design, only the first 8 entries are selectable.

| Code | Song |
|---|---|
| 000 | 1. Fur Elise - Beethoven |
| 001 | 2. Canon in D - Pachelbel |
| 010 | 3. Minuet in G major - Bach |
| 011 | 4. Turkish March - Mozart |
| 100 | 5. Nocturne in E flat - Chopin |
| 101 | 6. Waltz No. 2 - Shostakovich |
| 110 | 7. Nocturne in C sharp minor - Chopin |
| 111 | 8. Symphony No. 40 - Mozart |

## Software architecture

The firmware is structured around FreeRTOS tasks and timers rather than a single blocking loop for all behavior.

### Tasks and responsibilities

| Thread | Role |
|---|---|
| `update_lcd_leds_thread` | Updates the LCD status screen and controls the LED states |
| `polling_buttons` | Debounces buttons, reads key state, and handles song selection logic |
| `adjust_volume` | Monitors the potentiometer, applies volume changes, and prints UART instructions |
| `audio_task` | Configures the codec and starts the audio stream |

### Timing model

- The note progression is driven by a FreeRTOS software timer called the Ticker.
- The 5-second song confirmation timeout is implemented as a one-shot FreeRTOS software timer.
- The LCD is guarded by a mutex (`lcd_mutex`) to avoid concurrent drawing conflicts.
- The idle task calls `WFI` to sleep the CPU between interrupts, reducing power use.

### Audio implementation

Audio generation is performed in software rather than playing stored MP3 data. The code:

- uses the song definitions from `song_def.h`
- converts note values from milliseconds to frequency-like output values
- fills DMA buffers with tone samples
- updates the waveform in half/full-buffer interrupts
- drives the ES8388 codec through I2S3 in circular DMA mode

The note values are treated as periods in milliseconds, so the generated frequency is approximately `1000 / value` Hz. A value of 0 represents a rest.

## Pin mapping

| Part | Pin |
|---|---|
| Yellow LED | PB14 |
| Green LED | PA1 |
| Red LED | PA8 |
| Potentiometer (ADC3 channel 4) | PF6 |
| Keys: UP / DOWN / LEFT / RIGHT | PC5 / PC1 / PC0 / PC4 (active low) |
| USER_BUTTON (breadboard, to GND) | PA0 |
| UART (USART1, 115200 baud, via ST-Link USB) | PA9 (TX), PA10 (RX) |
| LCD (ST7789, FSMC 8-bit) | D0-D7 = PD14, PD15, PD0, PD1, PE7-PE10; RS = PD13 (A18); RD = PD4; WR = PD5; CS = PG10 (NE3); RST = PD3; BL = PF9 |
| Audio codec control (I2C2) | PF1 (SCL), PF0 (SDA), address 0x10 |
| Audio codec data (I2S3) | PC7 (MCK), PA15 (WS), PB3 (CK), PB5 (SD) |

The LEDs, potentiometer, and USER_BUTTON are mounted on a breadboard; most of the rest of the hardware is built into the RT-Spark board itself.

## Project structure

```text
Personal-MP3-Player-main/
├── README.md                    # Project overview and instructions
├── platformio.ini               # PlatformIO build configuration
├── fpu_link.py                  # Linker flags for Cortex-M4F floating point
├── include/                     # Firmware headers and course-provided song metadata
│   ├── FreeRTOSConfig.h
│   ├── lcd.h
│   ├── lcd_font.h
│   ├── songs.h
│   ├── song.h
│   ├── song_def.h
│   └── README
├── src/                         # Main firmware sources
│   ├── main.c                   # Embedded player logic, task setup, timing, audio
│   ├── lcd.c                   # LCD driver for ST7789 display
│   ├── songs.cpp               # C++ wrapper exposing song metadata to C firmware
│   └── ...
├── lib/                         # External libraries and FreeRTOS kernel
│   └── FreeRTOS-Kernel/
├── ref/                         # Reference files from the course handout
├── docs/                       # Project documentation and lab report
├── test/                       # PlatformIO unit test scaffolding
└── .vscode/                    # Editor configuration
```

### Important notes about the code organization

- `song.h` and `song_def.h` are part of the course-provided material.
- These files are written in C++, so they are only included from `src/songs.cpp`.
- The rest of the firmware remains in C to fit the STM32 HAL + FreeRTOS design.
- `songs.h` provides a simple C-compatible interface to the song metadata.

## Build and upload instructions

### Prerequisites

1. Install VS Code.
2. Install the PlatformIO extension for VS Code.
3. Install the appropriate ST-Link USB driver if Windows reports a driver issue.

### Build the project

1. Open the project folder in VS Code.
2. Ensure the PlatformIO environment is initialized.
3. Click the PlatformIO Build button (the checkmark icon) in the status bar.

### Upload to the board

1. Connect the RT-Spark board via ST-Link USB.
2. Click the PlatformIO Upload button (the arrow icon).
3. If the board is not detected or the ST-Link device appears missing, install ST's ST-Link drivers.

### Serial monitor

You can open the serial monitor at 115200 baud to view instructions and status output during startup.

### Build configuration

The project uses:

- `board = black_f407zg`
- `framework = stm32cube`
- `upload_protocol = stlink`
- Cortex-M4F floating point configuration via `-mfpu=fpv4-sp-d16 -mfloat-abi=hard`

This is configured in `platformio.ini`, and `fpu_link.py` applies the same floating-point flags at link time.

## Running the player

When the firmware boots:

1. The board initializes the LCD, LED pins, UART, ADC, I2C, and codec.
2. The system prints startup instructions over UART.
3. The current song list and controls are displayed to the user.
4. The player waits for key input and begins playback according to the selected song.

## Troubleshooting

### ST-Link connection errors

If Windows shows an error on the ST-Link device:

- install the ST-Link USB driver from STMicroelectronics
- reconnect the board
- retry upload

### No audio output

Check the following:

- codec power and I2C wiring
- I2S pin connections
- headphone output or speaker connection
- whether the device is stuck in paused state

### Song selection not working

Make sure:

- the keys are wired with the expected active-low behavior
- the USER_BUTTON is properly connected to ground
- debounce logic is operating as expected
- you are pressing the correct keys in the coded sequence

### Volume not responding

- confirm the potentiometer is connected to ADC3 channel 4 (PF6)
- verify the analog input and ADC initialization are correct
- inspect UART output to ensure the volume changes are being applied

## Differences from the original course handout

This implementation adapts the lab design to a practical embedded workflow:

- uses the onboard keys as the primary control interface
- treats the USER_BUTTON as an additional hardware button instead of a dedicated onboard key
- uses only the first 8 available songs from `song_def.h`
- uses single LEDs instead of a multi-color LED
- implements the project with PlatformIO and `main.c` rather than a CubeMX/CubeIDE project layout
- tunes tempo with a software scale rather than the original timing code from the course handout

## Credits and acknowledgements

- `song.h` and `song_def.h` are course-provided resources.
- The ES8388 startup register sequence is inspired by the RT-Thread ES8388 driver (Apache 2.0 license).
- The LCD font table was generated using DejaVu Sans Mono Bold, which is distributed under a free-use compatible license.

## Documentation and references

For deeper documentation, circuit details, and demonstration material, refer to the project page and related lab report:

-
- `docs/Laboratory Report/Laboratory Report - Personal MP3 Player.pdf`

## License and usage notes

This project is designed for academic and educational use as part of embedded systems coursework. The repository contains course material, firmware, and hardware-specific code for the RT-Spark board. Please respect the licensing terms of any third-party library components included in the project, especially the FreeRTOS kernel and the RT-Thread-derived ES8388 configuration ideas.
# bca182-laboratory2-mp3-player
