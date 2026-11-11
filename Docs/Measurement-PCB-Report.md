# SPECTER Measurement PCB — Project Report

## Purpose and scope

SPECTER is a prototype automated PCB fault-detection instrument using robotic probes. This report covers its original four-layer measurement and acquisition PCB, before the later cost-reduction redesign. The board is intended to support DC voltage and current measurement and an externally excited, impedance-related measurement path. The wider instrument includes separate signal-generation and power hardware, control software and robotic positioning; these are outside this repository.

## Architecture

The project README describes three conditioned measurement channels: a voltage path using OPA320 and LTC6910-1, a current path using a 50 milliohm shunt, INA188 and LTC6910-1, and an impedance-related path using OPA320 and LTC6910-1. Relays select the input path and route a programmable-gain output to a shared ADA4945-1 differential ADC driver and AD4003 ADC. The STM32H743 is intended to control acquisition and communication. The shared ADC means the three paths are not digitized simultaneously.

A protected external 12 V input feeds the board's 5 V and 3.3 V converters; a 1.8 V regulator is fed from 3.3 V. The detailed component references, power tree, signal flow and connector assignments are recorded in the [README](../README.md).

## Design files and views

- [KiCad project ZIP](../SPECTER-KiCad-Project.zip), including schematic, board, rules and local libraries. Extract it before opening the project.
- [Schematic](../Images/Schematic/SCBiS.svg), [front PCB](../Images/PCB/front.svg) and [back PCB](../Images/PCB/back.svg), exported from the included design.
- [Supplied 3D view](../Images/3D-view.png); its correspondence to this exact board revision has not been checked.

The included PCB SHA-256 matches the hash recorded in the supplied detailed README for the 24 October 2026 revision. Keep the local libraries and design rules together with the KiCad project. Standard KiCad 3D models are also required.

## Status and limitations

The wider prototype targets impedance measurements from 1 kHz to 100 kHz and ±2% magnitude accuracy after calibration, according to the supplied README. These are system goals, not demonstrated results for this board. The README also notes that the probe return nets are not internally tied to ground in this revision; the complete external return arrangement must be defined and checked.

This report summarizes the supplied project documentation and design; it is not a new electrical verification. Measurement ranges, accuracy, timing, power limits, functional behavior and fabrication readiness have not been independently validated for this package. The manufacturer's ADC maximum sampling rate is a component rating, not a demonstrated system rate. See the [README](../README.md) for manufacturer documentation and further limitations.

## Further work

Define and verify external probe connections, measurement ranges and calibration; develop and test firmware sequencing, settling and acquisition; and document measured system performance and fabrication checks before relying on the design.

