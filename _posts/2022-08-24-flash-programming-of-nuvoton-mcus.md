---
layout: post
title:  "Flash memory programming of the Nuvoton microcontrollers"
date:   2022-08-24 22:34:57 +0800
categories: 
tags: [MCUs, Nuvoton, NuMicro, Cortex-M]
---
### In-Circuit Emulation (ICE)
ICE enabled processor core registers and memory access through the external debug probe such like Nu-Link, ST-LINK, J-Link or something else. It helps developer to track the execution status of the microcontroller without braking the functionality of the target circuit. The common interface of the ICE is Serial Wire Debug Port (SW-DP or SWD) and JTAG-DP which spec in the ARM Debug Interface Architecture Specification. Nuvoton's chips usually provide the SWD as the ICE interface.

### Flash Memory Controller (FMC)
FMC is a interface between system memory and on-chip flash memory, it allow us to access flash memory via FMC registers which is already located in System Memory Map.

### Memory map
In most of the Nuvoton's chips, the Flash Memory Map is different from System Memory Map. The `Flash Memory Map` is used for the FMC to access flash memory. And the `System Memory Map` is what CPU looking through. 

### Remapping
The flash memory can be map into the System Memory Map, the map address is setup by Chip Booting Selection (CBS) in the config bits. For example, the Flash Memory Map of LDROM start at 0x00100000 it can be map to 0x00000000 in the System Memory Map by select `Boot from LDROM without IAP mode` in CBS.

### Chip Booting Selection (CBS)

### Flash memory organization
Most of the Nuvoton's chips has three different blocks of flash memory, inclueds APROM, LDROM and Data Flash. In generaly, `APROM` is used to store the program which provide to the target application, `LDROM` is used to store the bootloader which will execute before the application. The `Data Flash` is shared with APROM, it determin by config bits which user programmed, and it can be programmed through `ISP Commands` without unlocking the write permission of APROM.

### In-System Programming (ISP)
FMC provide `ISP Commands` include read, erase, program and remap. Accessing the FMC registers with corresponding commands and data via `firmware` or `ICE` can produce the flash memory programing. If you are doing APROM update by the firmware, it usually has a separate firmware called `bootloader` which located in LDROM to interact between FMC and the others peripheral such like UART, USB, CAN.

### In-Circuit Programming (ICP)
ICP utilize the memory access feature of ICE, produce the flash memory programing by `ISP Commands`. It needs a programming tool (software) to process the commands and data between host computer and FMC registers. Nuvoton provide NuMicro ICP Programming Tool which can be download from their website.

### In-Application Programming (IAP)
IAP brings more flexibility to the developer. If the IAP function is enabled by the CBS[0], the whole flash memory will appear in System Memory Map. This means CPU can executing the code in APROM, LDROM even the SRAM without rebooting the chip.

### References
- [ZaleYu/OpenOCD-Nuvoton](https://github.com/ZaleYu/OpenOCD-Nuvoton)
- [TRM_M0A21_M0A23_Series_EN_Rev1.02.pdf](https://www.nuvoton.com/export/resource-files/TRM_M0A21_M0A23_Series_EN_Rev1.02.pdf)