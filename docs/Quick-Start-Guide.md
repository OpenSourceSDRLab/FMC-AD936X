---
title: FMC-AD936X Quick Start Guide
product: FMC-AD936X
doc_id: OSSDRLAB-FMC-AD936X-QSG
version: 1.0
date: 2026-11-12
repo: https://github.com/OpenSourceSDRLab/FMC-AD936X
---

<!-- ossl:header -->
<p><a href="https://www.opensourcesdrlab.com"><img src="../images/OpenSourceSDRLab-banner.jpg" alt="OpenSourceSDRLab" width="100%"></a></p>

# FMC-AD936X Quick Start Guide

| Product | Document | Version | Date |
|---|---|---|---|
| FMC-AD936X | OSSDRLAB-FMC-AD936X-QSG | 1.0 | 2026-11-12 |
<!-- /ossl:header -->

## 1. Overview

The FMC-AD936X is an FPGA Mezzanine Card (FMC, LPC connector) built around the Analog Devices AD9361 or AD9363 RF transceiver (sold as FMC-9361 and FMC-9363). Its FMC signal assignment follows the Analog Devices AD-FMCOMMS2-EBZ board, so the FMCOMMS2 reference designs run on it. Compared with that board, each transmit output has an additional PGA-102+ amplifier, and the power-monitor chip and the FMC EEPROM are not fitted.

This guide shows how to run the bare-metal reference design on a Xilinx ZC706 carrier, check the received samples on the serial console, and rebuild the design from the sources in this repository.

## 2. What you need

| Item | Notes |
|---|---|
| FMC-AD936X board | |
| Carrier board | Xilinx ZC706 (XC7Z045), FMC **LPC** connector |
| SMA cable and attenuator | for the TX-to-RX loopback test; use an attenuator of at least 30 dB |
| USB cable | for the ZC706 USB-UART console |
| microSD card | only if you boot from the SD card |
| PC | Windows or Linux, with a serial terminal program |
| Prebuilt files | `FMC-AD936X_ZC706_prebuilt_Vivado2021.1.zip` from the [Releases](../../../releases) page |
| AMD/Xilinx Vivado and Vitis 2021.1 | <https://www.xilinx.com/support/download.html> (needed to program the board and to rebuild the design) |

## 3. Board connectors

| Connector | Function |
|---|---|
| TX1A, TX2A | transmit outputs (SMA), each with a PGA-102+ amplifier |
| RX1A, RX2A | receive inputs (SMA) |
| TX LO, RX LO | external local-oscillator inputs |
| EXT CLK | reference clock connector, on the same net as the on-board 40 MHz oscillator |
| JP5 | 20-pin header with the AD936x ENABLE, TXNRX, EN_AGC, CTRL_IN, CTRL_OUT and GPO signals |
| PWR | power LED |
| FMC (LPC) | LVDS data port, SPI and control signals, 12 V / 3.3 V / VADJ supplies from the carrier |

See `hardware/FMC-AD936X_Schematic.pdf` for the full schematic.

> **Tip:** the FPGA design uses 2.5 V I/O standards (LVDS_25, LVCMOS25) on the FMC connector. Keep the ZC706 VADJ supply at its default of 2.5 V.

## 4. Run the reference design (ZC706)

The reference application is the Analog Devices no-OS AD9361 example, configured for this board:

- TX and RX LO frequency 70 MHz, RF bandwidth 18 MHz, transmit attenuation 0.01 dB (maximum output power);
- the transmitter sends a sine tone from DDR memory through DMA (`DAC_DMA_EXAMPLE`);
- the receiver captures a block of samples through DMA (`ADC_DMA_EXAMPLE`) and prints 2048 I/Q samples of both receive channels on the serial console.

1. Switch the ZC706 off. Fit the FMC-AD936X on the ZC706 **LPC** FMC connector (J5).

2. Connect **TX1A** to **RX1A** through the attenuator. The transmit amplifier can overload the receiver if you connect the ports directly.

3. Connect the ZC706 USB-UART port to the PC and the ZC706 JTAG (USB) port to the PC, then switch the ZC706 on.

4. Open a serial terminal on the USB-UART port: **115200 baud, 8 data bits, no parity, 1 stop bit**.

5. Unpack `FMC-AD936X_ZC706_prebuilt_Vivado2021.1.zip` and load the design with one of these methods.

    **Over JTAG with Vitis 2021.1:** create the platform and application as described in section 5, then use *Run As > Launch Hardware*. Vitis programs the bitstream, initialises the processor system and runs the application.

    **From an SD card:** build a boot image with bootgen (part of Vitis) and the boot image description below, copy `BOOT.bin` to a FAT32-formatted microSD card, set the ZC706 boot-mode switch SW11 to SD boot (see the ZC706 user guide, UG954) and power-cycle the board.

        the_ROM_image:
        {
          [bootloader] fsbl.elf
          system_top.bit
          AD936X.elf
        }

        bootgen -arch zynq -image boot.bif -o BOOT.bin -w

6. The console shows the AD936x initialisation and interface-tuning messages, then a line starting with `DAC_DMA_EXAMPLE: address=`, followed by 2048 lines like

        ADC1_I=...,ADC1_Q=...,ADC2_I=...,ADC2_Q=...,

    `ADC1` is receive channel 1 (RX1A) and `ADC2` is receive channel 2 (RX2A). With the loopback cable on RX1A, the `ADC1` values show the received tone; you can copy them into a spreadsheet or a script to plot the waveform.

> **Tip:** the LO frequency, sample rates, bandwidths and gains are set in the `default_init_param` structure in `src/main.c`. The AD9363 (FMC-9363 boards) is specified for 325 MHz to 3.8 GHz; on these boards set the LO to a frequency in that range.

## 5. Reference designs in this repository

| Folder | Contents |
|---|---|
| `vivado/ZC706/FMC_AD936X_PL/` | FPGA design (PL): the Analog Devices HDL reference design `fmcomms2/zc706` (release `hdl_2021_r1`, Vivado 2021.1) with the HDL libraries it uses |
| `vivado/ZC706/FMC_AD936X_PS/` | Bare-metal application (PS): Analog Devices no-OS AD9361 driver and example (release `2021_R1`) configured for this board, and the hardware platform `system_top.xsa` |

The repository holds the sources. The prebuilt bitstream, `.xsa`, FSBL and application `.elf` are attached to the [Releases](../../../releases) page.

### Rebuild the FPGA design

The FPGA design is built with the Analog Devices make flow, which creates the Vivado project from Tcl scripts.

1. Install Vivado 2021.1 and put it on the `PATH`. On Windows, set up the build environment described in the Analog Devices HDL build guide: <https://wiki.analog.com/resources/fpga/docs/build>. Keep the repository in a short path (for example `C:\work`), because Vivado has problems with long Windows paths.

2. Build the project:

        cd vivado/ZC706/FMC_AD936X_PL/projects/fmcomms2/zc706
        make

    This builds the HDL libraries the design needs, then creates and implements the Vivado project `fmcomms2_zc706.xpr`.

3. The results are `fmcomms2_zc706.runs/impl_1/system_top.bit` and the hardware platform `fmcomms2_zc706.sdk/system_top.xsa`. You can also open `fmcomms2_zc706.xpr` in Vivado after the first build.

### Rebuild the application

1. In Vitis 2021.1, create a platform project from `system_top.xsa` (operating system `standalone`, processor `ps7_cortexa9_0`, with boot components so that the FSBL is generated).

2. Create an empty bare-metal C application named `AD936X` on that platform. Delete the generated `src` folder and import the folders `ad9361`, `api`, `AXI_Core`, `iio`, `include`, `src`, `util` and `xilinx` from `vivado/ZC706/FMC_AD936X_PS/AD936X/` into the application project. `src/lscript.ld` is the linker script.

3. In the application's C/C++ build settings, add these include directories (ARM v7 gcc compiler > Directories):

        ad9361
        api
        AXI_Core
        iio
        include
        src
        util
        xilinx

    The linker libraries are the Vitis defaults (`-Wl,--start-group,-lxil,-lgcc,-lc,--end-group`).

4. Build the **Release** configuration. Run it over JTAG from Vitis, or create `BOOT.bin` with the FSBL, bitstream and application as shown in section 4.

## 6. Support

- Product page: <https://opensourcesdrlab.com/products/opensourcesdrlab-fmc-936x-high-quality-development-board-card-with-lpc-interface-replacement-for-fmcomms2>
- Email: support@opensourcesdrlab.com
- Forum: <https://forum.opensourcesdrlab.com>
- Discord: <https://discord.gg/nnxY6DkmvT>

<!-- ossl:footer -->
---
<sub>© 2026 OpenSourceSDRLab. All rights reserved. · <a href="https://www.opensourcesdrlab.com">www.opensourcesdrlab.com</a></sub>
<!-- /ossl:footer -->
