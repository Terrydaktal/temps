# temps

Readable Linux hardware monitoring dashboard for temperatures and fan speeds.

This project wraps `lm-sensors` JSON output and hwmon/sysfs PWM data into a clearer terminal view than raw `sensors`.
```
Temperatures (15)
  Temp  | Unst  | Crit   | Load                       | Sensor                    | Device                     
  ------+-------+--------+----------------------------+---------------------------+----------------------------
  77.6C | 90.0C | 95.0C  | [###################.....] | Tctl                      | AMD Ryzen 9 9950X          
  77.0C | 90.0C | 95.0C  | [##################......] | AMD TSI 98h (Tctl smooth) | nct6686                    
  75.1C | 90.0C | 95.0C  | [##################......] | Tccd1                     | AMD Ryzen 9 9950X          
  64.8C | 75.0C | 85.0C  | [################........] | SSD Controller (Sensor 1) | WD_BLACK SN8100 2000GB     
  61.8C | 90.0C | 95.0C  | [###############.........] | Tccd2                     | AMD Ryzen 9 9950X          
  60.0C | 85.0C | 100.0C | [##############..........] | VRM (Thermistor 15)       | nct6686                    
  58.5C | 85.0C | 100.0C | [##############..........] | temp1                     | Realtek Ethernet Controller
  57.5C | 60.0C | 65.0C  | [##############..........] | temp1                     | DIMM Thermistor (SPD5118)  
  49.0C | 65.0C | 80.0C  | [############............] | System (Thermistor 14)    | nct6686                    
  47.9C | 79.8C | 84.8C  | [###########.............] | Composite                 | WD Blue SN570 1TB          
  47.9C | 89.8C | 93.8C  | [###########.............] | Composite                 | WD_BLACK SN8100 2000GB     
  46.9C | 70.0C | 80.0C  | [###########.............] | NAND (Sensor 2)           | WD_BLACK SN8100 2000GB     
  46.0C | 70.0C | 85.0C  | [###########.............] | temp1                     | MediaTek MT7921 Wi-Fi      
  44.0C | 83.0C | 90.0C  | [###########.............] | GPU Core                  | NVIDIA GeForce RTX 5060    
  39.5C | 80.0C | 90.0C  | [#########...............] | Socket (Thermistor 16)    | nct6686                    

Fan Speeds (3)
  RPM  | Load                              | Control | Fan           | Device 
  -----+-----------------------------------+---------+---------------+--------
  1608 | [################........]  65.1% | 166/255 | Chassis Fan 2 | nct6686
  1224 | [############............]  50.2% | 128/255 | Chassis Fan 1 | nct6686
  1004 | [############............]  50.2% | 128/255 | CPU Fan       | nct6686
```

## Requirements

- Linux with `lm-sensors` installed (`sensors` command available)
- Python 3.10+
- Access to `/sys/class/hwmon` (default on local system)

## Project Structure

```text
.
├── .gitignore
├── temps
├── README.md
└── sensor_thresholds.json
```

## Files

- `temps`
  - Executable Python CLI (no extension).
  - Inputs:
    - `sensors -j` output from `lm-sensors`
    - `/sys/class/hwmon/*` files for PWM control values and device identification
    - `sensor_thresholds.json` for fallback threshold rules
  - Outputs:
    - One-shot terminal dashboard
    - Live dashboard with in-place redraw (`--watch`)
  - Main features:
    - Temperature table (sorted hottest first)
    - Fan table with live RPM and PWM control percentage
    - Device name resolution (CPU model, NVMe model, nct6686, etc.)
    - Warn/Crit thresholds from real sensor data, with fallback overrides
    - Threshold-colored temperature bars:
      - Green: below warn
      - Orange: warn reached
      - Red: at or above 90% of crit

- `sensor_thresholds.json`
  - Threshold override database used when sensor limits are missing.
  - Rule matching uses regex for `device` and `sensor`.
  - `prefer_reported: true` means "keep real sensor values when present; only fill gaps."
  - `prefer_reported: false` forces the override value to replace sensor-provided values.

- `.gitignore`
  - Ignores Python cache artifacts.

## Data Pipeline

1. `temps` executes `sensors -j`.
2. It parses all `temp*_input` and `fan*_input` entries.
3. It resolves device names from hwmon sysfs metadata.
4. It reads PWM control channels (`pwmX`) from `/sys/class/hwmon` and maps them to `fanX`.
5. It loads `sensor_thresholds.json` and applies matching threshold fallback rules.
6. It renders terminal tables with aligned columns and load bars.
7. In watch mode, it redraws in place without flooding scrollback.

## Usage

Default behavior (live mode, 1-second refresh):

```bash
./temps
```

Run once (static snapshot):

```bash
./temps --once
```

Disable ANSI colors:

```bash
./temps --once --no-color
```

Live mode with custom refresh:

```bash
./temps --watch 1
# or
./temps --wait 1
```

## Live Mode Behavior

- Uses terminal alternate screen buffer for a clean dashboard view.
- Redraws in-place (no continuous scrolling output).
- Exit with `Ctrl+C`.
- Live mode is the default. Use `--once` for non-live output.

## Threshold Overrides

Edit `sensor_thresholds.json` to tune Warn/Crit values.

Each override supports:

- `name`: human-readable label
- `device_regex`: regex matched against resolved device name
- `sensor_regex`: regex matched against sensor label
- `warn`: warning threshold (Celsius)
- `crit`: critical threshold (Celsius)
- `prefer_reported`: whether to preserve real sensor-provided limits

Example rule:

```json
{
  "name": "Example Device Rule",
  "device_regex": "^Some Device$",
  "sensor_regex": "^Sensor 1$",
  "warn": 70.0,
  "crit": 85.0,
  "prefer_reported": true
}
```

## Notes

- Not all sensors export meaningful min/max/crit values.
- Some drivers report placeholders (for example `0`, `-273.15`, or unrealistic large sentinels).
- This tool filters invalid values and uses overrides to fill missing limits where appropriate.
