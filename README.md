# Gear Shifter USB Joystick (STM32F103C8x)

Firmware project for an STM32F103C8x ("Blue Pill") that emulates a USB game controller for an H-pattern gear shifter.

The original implementation was written for an ATmega32u4 in Arduino IDE using the `Joystick` library. This project is a full port to STM32CubeIDE / STM32 HAL with identical behavior:

* 2 analog inputs (X/Y axes of the shifter)
* 1 digital input (reverse lockout switch)
* 6 gear "gates" (positions) mapped to buttons
* Reverse gear with 2 distinct button states depending on the lockout switch
* USB HID Joystick device visible in Windows as a game controller
* Calibration over UART using a simple JSON protocol
* Thresholds stored persistently in Flash (EEPROM emulation)

---

## Features

* **MCU**: STM32F103C8x (Blue Pill or compatible)
* **USB**: Full-speed HID device, detected as a joystick with 7 buttons
* **Inputs**:

  * 2× analog channels (X, Y) for shifter axes
  * 1× digital input with pull-up for reverse lockout switch
* **Gear layout**:

  * Gates 0..5 → logical gears 1..6
  * Buttons 0..4 → gears 1..5
  * Button 5 → reverse (gear 6 gate + switch = LOW)
  * Button 6 → reverse (gear 6 gate + switch = HIGH)
* **Calibration**:

  * Triggered by UART command (`99`)
  * MCU prints live ADC readings while waiting for JSON
  * Thresholds provided as JSON and stored in Flash
  * Command `33` prints current thresholds over UART

---

## Hardware Overview

Typical wiring (can be adjusted in CubeMX / code to match your hardware):

* **X axis**: Potentiometer → ADC1 IN3 (e.g. PA3)
* **Y axis**: Potentiometer → ADC1 IN2 (e.g. PA2)
* **Reverse switch**: GPIO input with internal pull-up (e.g. PB0)
* **USB**: Native USB FS (PA11 = USB_DM, PA12 = USB_DP)

The exact pin mapping is configured in the `.ioc` file and can be adapted to your board.

---

## Project Structure

The project is a standard STM32CubeIDE/HAL project. Key files:

* `Core/Src/main.c`

  * Main application logic
  * ADC reads, gear selection, HID report generation
  * UART command parsing and calibration
  * Flash (EEPROM) emulation for thresholds
* `Core/Inc/main.h`

  * Basic project definitions and includes
* `USB_DEVICE/App/usbd_conf.c`, `USB_DEVICE/App/usbd_desc.c`, `USB_DEVICE/App/usbd_hid.c`

  * USB Device configuration and HID class
  * Custom HID report descriptor for a 7-button joystick
* `.ioc`

  * CubeMX configuration (clocks, GPIO, ADC, USART, USB Device)

---

## USB HID Report

The device exposes a **single-byte HID input report**:

* **Bits 0..4** → buttons for gears 1..5
* **Bit 5** → reverse gate selected *and* reverse switch = LOW
* **Bit 6** → reverse gate selected *and* reverse switch = HIGH
* **Bit 7** → unused (padding)

The matching HID report descriptor (in `usbd_hid.c`) defines a Generic Desktop Joystick with 7 buttons and 1 padding bit.

---

## Gear Detection Logic

The shifter uses pre-defined numeric ranges (thresholds) on the X/Y ADC values for each gear gate.

* ADC resolution: 12-bit, values 0–4095 (mapped logically to 0–1023 ranges used by the original Arduino code)
* For each gate `i` in `[0..5]` there are four thresholds:

  * `lowThresholdX[i]`, `highThresholdX[i]`
  * `lowThresholdY[i]`, `highThresholdY[i]`
* For a sample `(x, y)` to be considered inside gate `i`:

  ```c
  if (x >= lowThresholdX[i] && x <= highThresholdX[i] &&
      y >= lowThresholdY[i] && y <= highThresholdY[i]) {
      selectedGate = i;
  }
  ```

If no gate matches, no gear is selected and all buttons are released.

---

## Persistent Thresholds (Flash / EEPROM Emulation)

The original Arduino code stores thresholds in EEPROM. On STM32F1 this is emulated using a reserved Flash page near the end of memory.

A simple struct is stored in Flash:

```c
typedef struct {
    uint16_t magic;       // 0x00AB when initialized
    int16_t lowX[6];
    int16_t highX[6];
    int16_t lowY[6];
    int16_t highY[6];
} ThresholdStorage_t;
```

* If `magic` is invalid, default thresholds are used and written to Flash.
* On successful calibration, the struct is erased and re-written.
* The Flash page address is configured via `FLASH_PAGE_ADDR` and must match the MCU flash size (e.g. 0x0800FC00 for 64 KB).

> **Note**: If you use a different STM32F1 variant (e.g. with 128 KB Flash), adjust the last page address accordingly.

---

## Calibration over UART

Calibration is performed via UART1 (default 9600 8N1). The protocol closely mirrors the original Arduino implementation.

### Commands

* `99` → enter calibration mode
* `33` → print current threshold values

### Calibration Flow

1. Open a serial terminal on UART1 (e.g. USB-UART adapter connected to PA9/PA10).

2. Send `99` followed by any non-digit (e.g. `99\n`).

3. The MCU prints live ADC values every ~100 ms:

   ```text
   Entering Calibration Mode
   Send JSON terminated by 'x' (keys: lowThresholdX, highThresholdX, lowThresholdY, highThresholdY)
   Current Values - X: 512   Y: 742
   Current Values - X: 513   Y: 741
   ...
   ```

4. Prepare a JSON payload with thresholds, terminated by the character `x`. Example:

   ```json
   {"lowThresholdX":[0,0,340,370,540,545],
    "highThresholdX":[340,370,540,545,1023,1023],
    "lowThresholdY":[680,0,680,0,680,0],
    "highThresholdY":[1023,280,1023,280,1023,280]}x
   ```

5. Send the whole line to the MCU. On success you should see something like:

   ```text
   Received JSON string: {...}x
   Calibration successful. Exiting Calibration Mode.
   ```

6. Thresholds are now stored in Flash and automatically loaded on reset.

### Printing Thresholds

To check the currently used thresholds, send:

```text
33
```

The MCU responds with:

```text
Currently Loaded Threshold Values:
Gear 1: X (low - high), Y (low - high)
...
Gear 6: X (low - high), Y (low - high)
```

---

## Building and Flashing

### Requirements

* STM32CubeIDE (or STM32CubeMX + your preferred toolchain)
* ST-LINK/V2 (or compatible programmer/debugger)
* STM32F103C8x board (e.g. Blue Pill)

### Steps

1. Open the project in **STM32CubeIDE**.
2. Make sure the `.ioc` configuration matches your hardware:

   * ADC1 channels for X/Y
   * GPIO for reverse switch
   * USB Device FS enabled, class HID
   * USART1 enabled at 9600 8N1
3. Build the project (`Project → Build` or hammer icon).
4. Connect ST-LINK to the board and flash the firmware (`Run → Debug` or `Run → Run`).
5. After reset, connect the board to a PC via USB.

Your OS should enumerate a new **USB Game Controller / Joystick**. On Windows you can verify this in:

> Control Panel → Devices and Printers → (right click) Game Controller Settings

Pressing gears on the physical shifter should toggle the corresponding buttons in the test window.

---

## Porting Notes (from Arduino to STM32)

* The behavior of the original Arduino `Joystick` library is reproduced using a custom HID report descriptor and STM32 USB Device HID class.
* EEPROM has been replaced with a simple Flash page-based storage structure.
* `Serial` operations are implemented using `HAL_UART_Transmit` / `HAL_UART_Receive` with a small integer parser and a minimal JSON-like parser for arrays.
* ADC resolution differences (10-bit vs 12-bit) are accounted for by using the same nominal 0–1023 ranges in thresholds; you can adjust scaling if needed.

---

## Customization

You can customize:

* **Pin mapping**: change ADC channels / GPIO pins in `.ioc` and in `main.c` macros.
* **USB report**: extend the HID report to include axes if you want to expose X/Y as analog joystick axes.
* **Update rate**: currently reports are sent every ~100 ms (to mirror the original Arduino `delay(100)`); you can increase the frequency.

---

## License

This project is intended as firmware for a custom gear shifter USB adapter.
License can be adjusted as needed (e.g. MIT, BSD, or proprietary for client work).
