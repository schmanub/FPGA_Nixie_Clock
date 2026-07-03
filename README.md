# Nixie Clock
 
A GNSS-disciplined nixie tube clock built around an iCE40 UP5K FPGA and an RP2350 MCU, driving IN-12B nixie tubes and INS-1 neon indicators from a custom high-voltage supply.
 
## Overview
 
This project combines classic Soviet-era nixie tube display technology with a modern mixed-signal control architecture. An FPGA handles the real-time, safety-critical work: fast peak-current-mode control of two independent switching converters and the high-voltage display drive. An RP2350 MCU manages USB, timekeeping, and slower supervisory tasks like voltage setpoints and GNSS time discipline.
 
## System architecture
 
- **Power**: A flyback converter steps 5V up to 170V for the nixie tubes, alongside a boost converter generating a 12V rail for gate drive and level shifting. Both converters run fast, cycle-by-cycle peak-current-mode control implemented in FPGA fabric, with slower outer-loop setpoints computed by the MCU from onboard ADC readings. An FPGA-controlled load switch gates VUSB to both converters for clean, controlled power sequencing.
- **Control**: The iCE40 UP5K FPGA runs the two independent fast control loops, their gate drivers, and a display controller that talks to the MCU over I2C to relay the current time to the display hardware.
- **Display drive**: MC14504B level shifters bridge logic-level signals up to the HV5530 high-voltage shift registers, which directly drive the IN-12B nixie tubes and INS-1 neon indicators.
- **Timekeeping**: A GNSS receiver provides NMEA time/date data and a 1PPS reference. The RP2350 parses NMEA sentences, handles timezone logic, and disciplines a DS3231 RTC, giving the clock accurate, power-loss-tolerant timekeeping without depending on a network connection.
## Why this design
 
Splitting the fast and slow control domains keeps the safety-critical converter regulation immune to software timing jitter. The FPGA's peak-current loops respond within a single switching cycle, while the MCU only needs to update setpoints occasionally. This keeps the fabric footprint small and pushes flash programming, USB handling, and time-sync logic onto the RP2350, where they're far easier to develop and iterate on.
 
## Status
 
Actively in development. Schematic design, power supply simulation, and system architecture are in progress.
 
## License
 
TBD

```mermaid
flowchart LR
    USB["USB"]

    subgraph TIME["MCU + Timekeeping"]
        GNSS["GNSS receiver"]
        MCU["RP2350"]
        RTC["DS3231"]
    end

    subgraph CTRL["Control (iCE40 UP5K FPGA)"]
        PWM170["170V fast loop peak current mode"]
        PWM12["12V fast loop peak current mode"]
        GATE170["Gate driver 170V side"]
        GATE12["Gate driver 12V side"]
        DISPCTRL["Display controller I2C slave"]
    end

    subgraph PWR["Power supply"]
        LSW["Load switch"]
        FLY["Flyback converter 5V → 170V"]
        BOOST["Boost converter → 12V"]
        LVL["MC14504B level shifter"]
        HV["HV5530 shift registers"]
    end

    DISP["IN-12B nixies + INS-1 neons"]

    USB -- "VUSB" --> LSW
    USB -- "VUSB" --> GATE12
    USB -- "USB D+/D-" --> MCU

    LSW -- "VUSB (switched)" --> FLY
    LSW -- "VUSB (switched)" --> BOOST

    GNSS -- "NMEA data" --> MCU
    MCU -- "discipline time" --> RTC
    RTC -- "current time" --> MCU

    MCU -- "SPI flash programming" --> CTRL
    MCU -- "170V setpoint" --> PWM170
    MCU -- "12V setpoint" --> PWM12
    MCU -- "I2C: time to display" --> DISPCTRL
    CTRL -- "enable/disable" --> LSW

    PWM170 -- "switching cmd" --> GATE170
    PWM12 -- "switching cmd" --> GATE12
    GATE170 -- "gate drive" --> FLY
    GATE12 -- "gate drive" --> BOOST

    BOOST -- "12V rail" --> GATE170
    BOOST -- "12V rail" --> LVL
    FLY -- "170V rail" --> HV

    DISPCTRL --> LVL
    LVL --> HV
    HV --> DISP
```
