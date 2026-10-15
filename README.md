# Mini SMU - USB I-V Curve Tracer

A low-cost, USB-controlled source-measure unit. It sources a programmable voltage, measures the current through a device under test (diode, LED, resistor), and streams I-V data to a Python script that plots the characteristic curve.

Inspired by automating bench measurements on a Keysight B2912B SMU: this project brings the same source-and-measure workflow down to a custom 2-layer PCB.

![Block diagram](docs/block_diagram.png)

## Project status

| Item | Status |
|---|---|
| Firmware (RP2040, C) | Written, syntax-checked; **on-hardware testing: TODO** |
| Host script (Python) | Written, tested against built-in simulator |
| Schematic / PCB (KiCad) | **TODO** - add `hardware/kicad/` |
| Board bring-up and calibration | **TODO** |

> `docs/simulated_iv_curve.png` is generated from a software diode model (`--simulate`), **not** from the physical board. Replace it with real measurements once the board is built.

## Design targets

These are design goals, not measured results. See the results table below for what the board actually achieves.

| Parameter | Target |
|---|---|
| Source range | 0 to 3.3 V (12-bit DAC, ~0.8 mV/LSB) |
| Current measurement | +/-8.19 mA full scale, 0.25 uA resolution (10 ohm shunt, INA226) |
| Voltage readback | 1.25 mV/LSB (INA226 bus voltage) |
| Software compliance | 7.5 mA, output forced to 0 V if exceeded |
| Interface | USB CDC serial, ASCII commands |
| Board | 2-layer PCB, 3V3 rail, USB powered |

## Hardware

- **MCU:** RP2040 (Pico-style module or bare chip)
- **Source:** MCP4725 12-bit I2C DAC, buffered by a rail-to-rail op-amp
- **Sense:** 10 ohm 0.1% shunt, read by an INA226 (shunt voltage and DUT voltage over I2C)
- **Layout notes:** analog and digital sections separated, solid ground plane, decoupling at each IC, test points on 3V3, GND, DAC_OUT, V_DUT, SDA, SCL

Bill of materials: [`hardware/BOM.csv`](hardware/BOM.csv)

## Repository layout

```
mini-smu/
  firmware/   RP2040 firmware (pico-sdk, C)
  host/       Python CLI: sweeps, CSV export, plotting, simulator
  hardware/   BOM (add KiCad project, Gerbers, schematic PDF)
  docs/       Block diagram, plots, photos
```

## Firmware

Build with the [Raspberry Pi pico-sdk](https://github.com/raspberrypi/pico-sdk):

```bash
cd firmware
cp $PICO_SDK_PATH/external/pico_sdk_import.cmake .
mkdir build && cd build
cmake .. && make -j4
# hold BOOTSEL, plug in USB, copy mini_smu.uf2 to the RPI-RP2 drive
```

### Serial protocol (115200, newline-terminated)

| Command | Reply | Description |
|---|---|---|
| `ID?` | `MINI-SMU,fw1.0` | Identify |
| `SET <mV>` | `OK` | Set output voltage |
| `OFF` | `OK` | Output to 0 V |
| `MEAS?` | `<v_dut_mV>,<i_uA>` | One measurement |
| `SWEEP <start> <stop> <step> <settle_ms>` | CSV rows, then `END` | Voltage sweep (mV); aborts on compliance |

Errors are returned as `ERR,<reason>` (`DAC_I2C`, `INA_I2C`, `COMPLIANCE`, `INA_NOT_FOUND`, ...).

## Host script

```bash
cd host
pip install -r requirements.txt

# Real hardware
python smu.py --port /dev/ttyACM0 sweep --start 0 --stop 3000 --step 20 \
    --csv led.csv --plot led.png --title "Red LED"

# No hardware: built-in LED model (output labelled SIMULATED)
python smu.py --simulate sweep --start 0 --stop 3000 --step 20 --plot sim.png
```

## Results (fill in after bring-up)

Compare against a multimeter or bench SMU at several points.

| Test point | Reference | Mini SMU | Error |
|---|---|---|---|
| 1.000 V output | | | |
| 2.000 V output | | | |
| 1 k ohm resistor, 1 mA | | | |
| Red LED forward voltage at 2 mA | | | |

Measured noise (std dev of 100 readings): `TODO`
Offset after calibration: `TODO`

![Board photo](docs/board_photo.jpg)
*(add a photo of your assembled board here)*

## Known limitations

- Single quadrant (0 to 3.3 V, positive current only in the default design)
- Current accuracy depends on shunt tolerance and INA226 offset; calibrate per board
- Software compliance only, with no hardware current limiter

## Future work

- Four-quadrant operation with a bipolar supply rail
- Hardware current limit
- Multiple current ranges via switched shunts
- Per-board calibration stored in flash

## License

MIT
