---
layout: post
title:  "Flash memory programming of the Nuvoton microcontrollers"
date:   2022-08-24 22:34:57 +0800
categories: 
tags: [MCUs, Nuvoton, NuMicro, Cortex-M]
---
### In-Circuit Emulation (ICE)
ICE enabled processor core registers and memory access through the external debug probe such like Nu-Link, ST-LINK, J-Link or something else. It helps developer to track the execution status of the microcontroller without braking the functionality of the target circuit. The common interface of the ICE is Serial Wire Debug Port (SW-DP or SWD) and JTAG-DP which spec in the ARM Debug Interface Architecture Specification. Nuvoton's chips usually provid the SWD as the ICE interface.

### Flash Memory Controller (FMC)
FMC is a interface between system memory and flash memory, it allow us to access `on-chip NOR flash` via FMC registers which is located in System Memory Map. 

### Memory map
In most of the Nuvoton's chips, the Flash Memory Map is different from System Memory Map. The `Flash Memory Map` is used for the FMC to access on-chip NOR flash. And the `System Memory Map` is what CPU looking through.

### Flash memory organization
Most of the Nuvoton's chips provid three different blocks of flash memory, inclueds APROM, LDROM and Data Flash. In generaly, `APROM` is used to store the program which provid the target application, `LDROM` is used to store the bootloader which will execute before the application. The `Data Flash` is shared with APROM which can be determin by user in config bits, and it can be programmed through FMC without unlocking the write permission of APROM.

### In-Circuit Programming (ICP)
ICP utilize the memory access feature of ICE, produce the flash memory programing by `operating the FMC registers`. It needs a programming tool to transport the corresponding command and data between FMC. Nuvoton provid NuMicro ICP Programming Tool which can download from their website.

### In-System Programming (ISP)
ISP also programed the flash memory by operating the FMC registers. The difference of ISP is, it `operate the FMC registers by firmware`. If you are doing APROM firmware update via the ISP, it usually has a separate firmware which located in LDROM to interact between FMC and the others peripheral such like UART, USB, CAN. 

### In-Application Programming (IAP)
IAP
