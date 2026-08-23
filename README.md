# STM32F407VE Oscilloscope

A digital oscilloscope built on an STM32F407VE with a 320x240 parallel TFT.
Timer-triggered ADC sampling, DMA to a double-buffered ring, edge triggering,
live rendering. Built to learn the STM32 platform, DMA and analogue front end
design.

**Status:** in progress. Nothing works yet.

## Hardware
- STM32F407VE "black board", 8 MHz HSE
- 3.2" 320x240 TFT, 16-bit parallel via FSMC, controller TBC
- ST-Link V2 (clone)
- Single-supply bench PSU — front end will be single-rail

## Log
See `LOG.md`.