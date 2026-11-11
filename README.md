# SPECTER Measurement PCB

A four-layer measurement and acquisition PCB developed for **SPECTER**, a prototype automated PCB diagnostic instrument using robotic probes.

This is the **original design before the cost reductions**: STM32H743, AD4003, ADA4945-1 and three programmable-gain measurement channels.


![Supplied 3D view of the measurement PCB](Images/3D-view.png)

## Design views and project files

- [Schematic](Images/Schematic/SCBiS.svg)
- [PCB front view](Images/PCB/front.svg) | [PCB back view](Images/PCB/back.svg)
- [Project report](Docs/Measurement-PCB-Report.md)
- [Download KiCad project](SPECTER-KiCad-Project.zip)

Download and extract the KiCad project ZIP, then open `SCBiS.kicad_pro` in KiCad 10. Keep the extracted files and local libraries together. Standard KiCad 3D models are also required. The supplied 3D image has not been checked against this exact board revision; the schematic and PCB SVGs were exported from the included files.
## Purpose

The board receives signals from the probe system, provides input conditioning and programmable gain, selects a measurement path and digitizes its response. An STM32 controller handles channel control, acquisition timing, processing and communication with the rest of SPECTER.

The complete instrument also includes a separate power/signal-generation subsystem, PC/Raspberry Pi software and robotic probe positioning. Those subsystems are outside this measurement PCB.

## Hardware architecture

| Block | Components | Function |
| --- | --- | --- |
| Controller | U1 STM32H743ZIT6 | Arm Cortex-M7 controller for acquisition and system communication |
| Voltage channel | U4 OPA320, U5 LTC6910-1 | Input buffering and programmable gain |
| Current channel | U17 FC4L16R050FER, U6 INA188, U7 LTC6910-1 | Shunt sensing, instrumentation amplification and programmable gain |
| Impedance-related channel | U10 OPA320, U11 LTC6910-1 | Buffered input and programmable gain for the externally excited measurement path |
| Input switching | K1, K2, K3 | Relay selection of voltage, current and impedance-related input paths |
| Output switching | K4, K5, K6 | Relay selection of the PGA output sent to the shared ADC driver |
| Current-path connection | K7 | Connects the INA188 output to the current PGA input |
| Relay drivers | U2, U3 ULN2003A | Drive relay coils from control signals |
| PGA control buffers | U12, U13, U14 SN74AHCT125 | Buffer STM32 gain-control signals into the 5 V logic domain |
| Differential ADC driver | U21 ADA4945-1 | Drives the ADC's differential inputs |
| ADC | U15 AD4003BRMZ | 18-bit differential SAR conversion |
| Reference | U8 ADR4525BRZ | Nominal 2.5 V precision reference |
| Reference-related buffers | U9, U18 OPA320 | ADC-driver common-mode and INA188 reference circuitry |
| Input supply protection | U27 MAX17608 | Protected supply entry |
| Supply conversion | PS1 TPS62163, U22 TPS62162, U20 TPS7A2018 | 5 V, 3.3 V and 1.8 V supplies |
| Buck inductors | L2 XGL5030-222MEC, L1 XGL4020-222MEC | Inductors for the 5 V and 3.3 V converters |

The AD4003 device supports up to 2 MSPS according to its datasheet. That device rating is **not a demonstrated sampling rate for this complete board**; firmware timing, interface configuration and analog settling must also be established.

## Measurement flow

```mermaid
flowchart TD
    PROBE["J6.1: probe signal"] --> K1["K1: voltage input"]
    PROBE --> K2["K2: current input"]
    PROBE --> K3["K3: impedance-related input"]

    K1 --> VP["Protection / scaling network"]
    VP --> VBUF["U4 OPA320"]
    VBUF --> VPGA["U5 LTC6910-1"]
    VPGA --> K4["K4"]

    K2 --> SHUNT["U17: 50 milliohm shunt"]
    SHUNT --> INA["U6 INA188"]
    INA --> K7["K7"]
    K7 --> IPGA["U7 LTC6910-1"]
    IPGA --> K5["K5"]

    K3 --> ZP["Protection network"]
    ZP --> ZBUF["U10 OPA320"]
    ZBUF --> ZPGA["U11 LTC6910-1"]
    ZPGA --> K6["K6"]

    K4 --> FDA["U21 ADA4945-1 differential driver"]
    K5 --> FDA
    K6 --> FDA
    FDA --> RC["ADC input resistor / capacitor network"]
    RC --> ADC["U15 AD4003"]
    ADC -->|"Serial samples"| STM["U1 STM32H743"]
    REF["U8 ADR4525 reference"] --> ADC
    STM -.->|"Via U12 / U13 / U14"| VPGA
    STM -.->|"Via U12 / U13 / U14"| IPGA
    STM -.->|"Via U12 / U13 / U14"| ZPGA
```

Each measurement path has its own conditioning and programmable-gain stage, but the differential driver and ADC are **shared**. K4-K6 select which PGA output reaches the driver. This revision uses relays for that selection; it does not contain the TMUX1204 introduced in the later budget board.

Separate front ends do not provide simultaneous digitization. Firmware must coordinate relay states, gain changes, settling and conversion timing. The current-path relay K7 is driven through a U3 channel whose input is connected to the 5 V power-good net in this saved revision.

## Power tree

```text
J4: external 12 V input
  |
  +-- U27 input protection --> protected +12V
                                |
                                +-- PS1 + L2 --> 5 V
                                |
                                +-- U22 + L1 --> 3.3 V --> U20 --> 1.8 V
```

The protected 12 V rail also supplies the INA188. The 5 V rail supplies the PGAs, analog driver, buffers and relay circuitry. The STM32 and ADC digital interface use 3.3 V; the AD4003 analog supply uses 1.8 V.

The 3.3 V converter is supplied from the protected input independently of the 5 V converter. USB VBUS is connected to detection circuitry rather than the board's operating 5 V rail.

## Connectors

Use KiCad pad numbers to establish orientation, rather than assuming left and right from a photograph.

| Connector | Purpose | Saved pin assignments |
| --- | --- | --- |
| J1 | ST-LINK / SWD | 1: 3.3 V reference; 2: SWDIO; 3: GND; 4: SWCLK; 5: NRST; 6: GND |
| J2 | ESP32 UART link | 1: STM32 TX; 2: STM32 RX |
| J3 | Raspberry Pi UART link | 1: STM32 TX; 2: STM32 RX |
| J4 | 12 V input from the power subsystem | 1: GND; 2: positive input |
| J6 | Probe connection | 1: `o/p_probe`; 2: `return_probe` |
| J7 | Current/probe-side connection | 1: `crnt_to_probe`; 2: `crnt_return` |
| J8 | USB-C communication | USB D-/D+ to the STM32, with VBUS detection |

J2 and J3 are two-wire signal connectors, not ESP32 or Raspberry Pi power outputs. Cross TX to RX and provide the required common-ground connection separately.

**The older probe-return arrangement differs from the budget board.** In this saved PCB, `return_probe` and `crnt_return` each connect only to their respective connector pad; they are not internally tied to GND. The complete external return wiring must therefore be defined and checked before using the measurement paths.

## Firmware responsibilities

- Configure the MCU clocks, communication interfaces and ADC conversion timing.
- Control input and output relays with appropriate mode sequencing.
- Set each LTC6910 programmable-gain code through the AHCT buffers.
- Allow the complete analog chain to settle after switching or gain changes.
- Read ADC samples and apply calibration and measurement processing.
- Exchange commands and results through the selected UART or USB interface.

SWD through J1 provides a programming and debugging interface. USB operation requires the appropriate application firmware or a supported system-bootloader configuration; the PCB alone does not implement a USB command protocol.

## Prototype status and scope

This repository description documents the original hardware architecture. It does not establish measured accuracy, successful acquisition at the ADC's maximum rate, immunity to interference or complete fault-detection performance.

The wider SPECTER prototype targets a controlled fault PCB. The intended impedance frequency range is 1 kHz-100 kHz, with a magnitude-accuracy target of +/-2% after calibration. These are system goals, not verified results for this revision.

Initial multimeter readings and assembly photographs from the later cost-reduced board do **not** validate this older design. End-to-end checks must cover the probe returns, input ranges, gain settings, reference levels, ADC-driver behavior, power sequencing and firmware before reporting performance.

## PCB and source files

- Four copper layers.
- Original STM32H743 / AD4003 / ADA4945-1 architecture.
- Seven relay channels and three LTC6910 programmable-gain stages.
- KiCad project: `SCBiS.kicad_pro`.
- Schematic: `SCBiS.kicad_sch` and the additional schematic sheets supplied with the project.
- Board: `SCBiS.kicad_pcb`.

Keep the project-local footprint and symbol libraries with the source files. The reference for this README is the saved 24 October 2026 routing/DFM revision of the original circuit, before the cost-reduction redesign.

```text
PCB SHA-256: b371d0481f8071126b6a58f23fe88a301be0a6cb624ea9d2b35b959b8cf2d500
```

## Manufacturer documentation

- [STM32H743 — controller datasheet](https://www.st.com/resource/en/datasheet/stm32h743zi.pdf)
- [AD4003 — ADC specifications, pin functions and serial interface](https://www.analog.com/media/en/technical-documentation/data-sheets/AD4003-4007-4011.pdf)
- [ADA4945-1 — fully differential ADC driver](https://www.analog.com/media/en/technical-documentation/data-sheets/ada4945-1.pdf)
- [LTC6910 family — digitally controlled programmable-gain amplifiers](https://www.analog.com/en/products/ltc6910.html)
- [ADR4525 — reference family datasheet](https://www.analog.com/media/en/technical-documentation/data-sheets/ADR4520_4525_4530_4533_4540_4550.pdf)
- [INA188 — instrumentation amplifier datasheet](https://www.ti.com/lit/ds/symlink/ina188.pdf)
- [OPA320 — operational amplifier datasheet](https://www.ti.com/lit/ds/symlink/opa320.pdf)
- [SN74AHCT125 — logic buffer datasheet](https://www.ti.com/lit/ds/symlink/sn74ahct125.pdf)
- [ULN2003A — relay-driver array datasheet](https://www.ti.com/lit/ds/symlink/uln2003a.pdf)
- [TPS6216x — buck converter datasheet](https://www.ti.com/lit/ds/symlink/tps62163.pdf)
- [TPS7A20 — LDO documentation](https://www.ti.com/product/TPS7A20)
- [MAX17608 — input protection datasheet](https://www.analog.com/media/en/technical-documentation/data-sheets/MAX17608-MAX17610.pdf)


