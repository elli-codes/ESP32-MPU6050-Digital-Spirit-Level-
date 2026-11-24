
# ESP32 + MPU6050 Digital Spirit Level ( From the Datasheet)

A digital spirit level built with an **ESP32** and an **MPU6050** accelerometer/gyroscope, written **without any MPU6050 library**. Every register is read and written directly over I2C, using only the official datasheet and Register Map.

Tilt the board left or right and one of two LEDs turns on to show the direction.
## What you will learn

- How to choose the right document: **Product Specification** vs **Register Map**
- How I2C works (SDA, SCL, master/slave, ACK) and how the MPU6050 address is set by **AD0**
- How to scan the I2C bus
- How to wake the sensor from sleep (`PWR_MGMT_1`)
- How to set the accelerometer and gyroscope ranges
- How to read 14 bytes of raw data in one burst and combine High/Low bytes into signed 16-bit values
- How to convert raw values to **g**, **°/s** and **°C**
- How to use accelerometer data to detect tilt

---

## Components

| Part | Qty |
|---|---|
| ESP32 DevKit | 1 |
| MPU6050 module (GY-521) | 1 |
| LED (red) | 1 |
| LED (blue) | 1 |
| Resistor 220 Ω | 2 |
| Breadboard | 1 |
| Jumper wires | several |
| USB cable | 1 |

---

## Wiring

| MPU6050 (GY-521) | ESP32 |
|---|---|
| VCC | **3V3** (not 5 V) |
| GND | GND |
| SDA | GPIO21 |
| SCL | GPIO22 |
| AD0 | GND (I2C address = `0x68`) |
| XDA, XCL, INT | not connected |

| LED | ESP32 |
|---|---|
| Red LED (via 220 Ω resistor) | GPIO5 |
| Blue LED (via 220 Ω resistor) | GPIO18 |

LED cathodes (short leg) go to GND.



> **Why 3.3 V?** The datasheet gives the supply range as 2.375 V to 3.46 V (Product Specification, p. 8), and the ESP32 GPIOs are 3.3 V logic.

---

## Datasheets used

- [MPU-6000/MPU-6050 Product Specification (PS-MPU-6000A)](https://cdn.sparkfun.com/datasheets/Components/General%20IC/PS-MPU-6000A.pdf)
- [MPU-6000/MPU-6050 Register Map and Descriptions (RM-MPU-6000A)](https://cdn.sparkfun.com/datasheets/Sensors/Accelerometers/RM-MPU-6000A.pdf)

---

## Step by step

### Step 1: Datasheet overview and I2C scan

| Question | Answer | Where |
|---|---|---|
| Which protocol? | I2C only (the MPU6000 also has SPI) | Product Spec, comparison table |
| Supply voltage? | 2.375 V to 3.46 V | Same table, VDD row |
| I2C address? | `b110100X`, X = AD0. AD0 low = `0x68`, high = `0x69` | Product Spec, section 9.2 |

Run an I2C scanner to confirm the sensor answers. With AD0 on GND, expect `0x68`.

> The scanner code is based on the example from [Random Nerd Tutorials](https://randomnerdtutorials.com/esp32-i2c-communication-arduino-ide/).

**WHO_AM_I** (register `0x75`, read-only, default `0x68`) is used to *confirm the chip's identity*. It does **not** find the address. You need the address first, then you read WHO_AM_I to verify who answered.

### Step 2: Wake up the sensor

After power-up the MPU6050 is in **sleep mode**. In `PWR_MGMT_1` (`0x6B`):

| Bit | Name | Value | Meaning |
|---|---|---|---|
| 7 | DEVICE_RESET | 0 | no reset |
| 6 | SLEEP | 0 | awake |
| 5 | CYCLE | 0 | normal mode |
| 3 | TEMP_DIS | 0 | temperature sensor on |
| 2:0 | CLKSEL | 001 | PLL with X-axis gyroscope reference |

Result: `0b00000001` = **`0x01`**. The datasheet strongly recommends a gyroscope clock over the internal oscillator for better stability.

```cpp
Wire.beginTransmission(0x68);
Wire.write(0x6B);   // PWR_MGMT_1
Wire.write(0x01);   // SLEEP = 0, CLKSEL = 1
Wire.endTransmission();
```

We read the register back to confirm the write worked.

### Step 3: Configure the measurement ranges

| Register | Address | Value | Range |
|---|---|---|---|
| `ACCEL_CONFIG` | `0x1C` | `0x10` | ±8 g |
| `GYRO_CONFIG` | `0x1B` | `0x10` | ±1000 °/s |

Bits 4 and 3 select the range (`AFS_SEL` / `FS_SEL`). Here they are `10` = 2.

| Select | Accelerometer | Sensitivity | Gyroscope | Sensitivity |
|---|---|---|---|---|
| 0 | ±2 g | 16384 LSB/g | ±250 °/s | 131 LSB/(°/s) |
| 1 | ±4 g | 8192 LSB/g | ±500 °/s | 65.5 LSB/(°/s) |
| **2** | **±8 g** | **4096 LSB/g** | **±1000 °/s** | **32.8 LSB/(°/s)** |
| 3 | ±16 g | 2048 LSB/g | ±2000 °/s | 16.4 LSB/(°/s) |

(From Product Specification sections 6.1 and 6.2.) Sensitivity is simply `32768 / range`, so a larger range means lower resolution.

```cpp
Wire.beginTransmission(0x68);
Wire.write(0x1C);   // ACCEL_CONFIG
Wire.write(0x10);   // ±8 g
Wire.endTransmission();

Wire.beginTransmission(0x68);
Wire.write(0x1B);   // GYRO_CONFIG
Wire.write(0x10);   // ±1000 °/s
Wire.endTransmission();
```

### Step 4: Read raw data (14 bytes)

Starting at `0x3B`, the sensor outputs 14 bytes in this order:

| Registers | Data |
|---|---|
| `0x3B` to `0x40` | Accelerometer X, Y, Z (High, Low each) |
| `0x41` to `0x42` | **Temperature** |
| `0x43` to `0x48` | Gyroscope X, Y, Z (High, Low each) |

Each value is a **signed 16-bit** number (two's complement), built as `(High << 8) | Low` and stored in an `int16_t`.

You give only the **start address**; the sensor automatically moves to the next register after each byte. You choose how many bytes to read with `requestFrom(0x68, 14)`.

> ⚠️ **Common bug:** the temperature bytes sit in the middle. If you skip them, the gyroscope values are wrong and no error is shown.

```cpp
Wire.beginTransmission(0x68);
Wire.write(0x3B);              // start address only
Wire.endTransmission(false);   // repeated START, no STOP
Wire.requestFrom(0x68, 14);    // read 14 bytes
```

### Step 5: Convert to real units

```cpp
float axg = ax / 4096.0;          // g
float gxd = gx / 32.8;            // °/s
float tempC = temp / 340.0 + 36.53;   // °C
```

Use `4096.0` (a float), not `4096`. Integer division would drop the decimals.

Sanity checks:

- **Flat on the table:** `az` is close to 1 g, `ax` and `ay` close to 0.
- **Total acceleration:** `sqrt(ax² + ay² + az²)` stays near 1 g in any orientation when the board is still.
- **Gyroscope at rest** is not exactly 0. A small offset is normal (the datasheet allows up to ±20 °/s zero-rate output).
- **Temperature** (around 40 °C) is the chip's die temperature, not the room temperature.

### Step 6: Build the digital spirit level

The accelerometer measures gravity along its own axes. When the board tilts, gravity moves between axes, so one axis grows positive or negative.

```cpp
float threshold = 0.2;

if (ayg > threshold) {
  digitalWrite(5, HIGH);
  digitalWrite(18, LOW);
}
else if (ayg < -threshold) {
  digitalWrite(5, LOW);
  digitalWrite(18, HIGH);
}
```

- The threshold of `0.2 g` (about 11°) is a dead zone, so noise and the small offset do not flip the LEDs.
- There is no final `else`, so the **last LED stays on** until you tilt to the other side.

> ⚠️ **Important note: which axis to use**
> Which direction counts as "left/right" (and which LED turns on) depends on how the sensor is mounted, because the accelerometer measures gravity relative to **its own axes**, not relative to your breadboard or your hand.
> In my setup:
> - Forward/back tilt → `axg`
> - Left/right tilt → `ayg`
>
> If your LEDs do not react, or react to the wrong movement, print `axg` and `ayg` on the Serial Monitor, tilt the board, and see which value changes. Use that axis in your code. To reverse the direction, swap the two LED pins.

---

## Final code

The full sketch is in [`05_tilt_led/`](05_tilt_led/). It combines all steps: wake up, configure ranges, burst read, convert, and control the LEDs.

Open the Serial Monitor at **115200 baud** to see `axg` and `ayg`.

---

## Repository structure

```
├── README.me
├── 01_i2c_scanner/
├── 02_wake_UP_config/
├── 03_set_ranges/
├── 04_final_code/
└── LICENSE
```

---

## Troubleshooting

| Problem | Check |
|---|---|
| Scanner finds no device | Power on 3V3, GND shared, SDA/SCL not swapped, GPIO21/22 used in `Wire.begin(21, 22)` |
| Scanner shows `0x69` | AD0 is high; connect it to GND for `0x68` |
| All values are 0 or constant | The sensor is still asleep; check the `PWR_MGMT_1` write |
| Gyroscope values look wrong | The temperature bytes (`0x41`, `0x42`) were skipped |
| Values are half or double what you expect | Sensitivity does not match the configured range |
| LEDs do nothing | Wrong axis (print `axg` and `ayg`), or threshold too high |
| LED flickers | Raise the threshold or add averaging |

---

## Next steps

- Offset calibration for the gyroscope and accelerometer
- Roll and pitch angles with `atan2`
- A third LED for "level" (both axes near zero)
- Complementary filter (accelerometer + gyroscope)
- Interrupts (`INT` pin) and the FIFO buffer

---

## Credits

- I2C scanner based on the example by [Rui Santos, Random Nerd Tutorials](https://randomnerdtutorials.com/esp32-i2c-communication-arduino-ide/)
- Datasheets by InvenSense (TDK)

## License

MIT License. See [LICENSE](LICENSE).

> 🎥 **Video tutorial:** `[ https://youtu.be/TbHxijNXRMk ]`
