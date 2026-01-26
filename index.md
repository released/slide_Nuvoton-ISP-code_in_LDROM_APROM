[return to index](https://released.github.io/)

<a id="article_top"></a>

# Nuvoton M2A23 – UART ISP Training Custom Flow

> This training material is written **to match the actual reference project**
> **M2A23BSP_ISP_UART_APROM**, including its **custom ISP flow split into `isp_config.c`**.
>

---


## Reference Project

This training material is based on the **below reference project**:

- https://github.com/released/M2A23BSP_ISP_UART_APROM

All memory layout, ISP flow, CRC strategy, and tool settings described below are
**directly derived from this repository**.


## Agenda

* System overview Flash layout + dual UART
* Boot code LDROM + APROM-end flow — **custom ISP flow**
* Application code flow
* Build & image generation
* Tool settings ICP / ISP
* Runtime notes & pitfalls

---

<a id="article_overview"></a>

## 1. System overview

### Flash allocation actual project


> **Important !!!** 
> **Address is configurable**  
> - `APROM Application` start / size  
> - `APROM Boot extension` base address example: `0x1E000`  
>
> These addresses are **project-dependent** and **NOT fixed by hardware**.  
> **must** adjust them according to:
> - Application code size requirement
> - Bootloader code feature size
>
> The values used in this training `0x1E000`, `0x1DFFC` are **one validated reference only**.


| Region | Address | Size | Purpose |
|------|--------|------|--------|
| APROM Application | `0x0000_0000 ~ 0x0001_DFFF` | `0x1E000` | app code |
| APROM checksum | `0x0001_DFFC` | 4 bytes |app code checksum address (CRC32) |
| Boot code ext in APROM | `0x0001_E000 ~ 0x0001_FFFF` | 8 KB | boot code extension |
| Boot code inLDROM | `0x0010_0000 ~ 0x0010_0FFF` | 4 KB | boot code |

![](img/FLASH_calculate.jpg)

### UART assignment hard separation

| UART | Function | Used in |
|----|----|----|
| UART0 PB12/PB13 | ISP protocol | Boot only , to upgrade app code|
| UART1 PA8/PA9 | printf / progress log | Boot + App |

---

<a id="article_boot_flow"></a>

## 2. Boot code flow LDROM + APROM-end

### Source-level structure important

```
main.c
 └─ main
    ├─ SYS_Init
    ├─ ISP_Init
    ├─ ISP_check_app
    └─ while1
        └─ ISP_process

isp_config.c   ← ★ Custom ISP state & policy
isp_user.c     ← UART RX / CMD handler
```

### Boot code flow

```mermaid
flowchart TD
    A[Reset / Power-on] --> B[SYS_Init + UART Init UART0=ISP, UART1=Log]
    B --> C[ISP_Init 
    FMC_Open + ISP Enable]
    C --> D[ISP_check_app]
    D --> E{Verify app CRC32 APROM 0..size-4 vs last word}
    E -->|YES| F[Jump to APROM VECMAP=APROM start CPU reset]
    E -->|NO| G[Stay in bootloader]
    G --> H[ISP_process]
    H --> I[CMD_CONNECT?]
    I -->|YES| J[Receive packet 64B ParseCmd]
    J --> K[Execute command Update/Erase/Run/Reset...]
    K -->|CMD_UPDATE_APROM| L[WriteData]
    L --> M[Update progress by UART1 log]
    M --> H    
    K-->|FINISH : CMD_RUN_APROM| N[SYS_ResetChip restart boot]
```

![](img/LDROM_upgrade_finish.jpg)


### Key point training emphasis

* **Protocol parsing and policy are separated**
  * `isp_user.c` → packet handling
  * `isp_config.c` → CRC check / boot decision / ISP behavior
* UART logging is **post-action only**, never **execute printf** during RX parsing

---

<a id="article_app_flow"></a>

## 3. Application code flow

```mermaid
flowchart TD
    A[App Reset Vector @ 0x0000_0000] --> B[System init peripherals, tasks]
    B --> C[Normal run]
    C --> D{Enter update mode? button/command/flag}
    D -->|Yes| E[Erase checksum @ 0x1DFFC]
    D -->|NO|C
    E --> F[SYS_ResetChip]
    F --> G[return to Boot
    compare checksum CRC FAIL → ISP mode]
```

### Practical triggers from reference

* Press **'1'** → erase checksum (under app code)

![](img/APROM_erase_checksum.jpg)


* Press **'Z' / 'z'** → reset to LDROM

![](img/APROM_press_Z_to_LDROM.jpg)


* Press **nRESET** on EVM

![](img/APROM_press_nRESET_to_LDROM.jpg)

---

<a id="article_build"></a>


### Scatter file in Boot code (uart_iap.sct – MUST use as-is)

Boot code project **uses a single scatter file**: `uart_iap.sct`.

- Adjust **only the address/size macros** if project memory layout changes

The linker layout for:
- LDROM
- APROM boot extension

is **centrally defined in `uart_iap.sct`** to guarantee consistency between
bootloader and application images.

![](img/LDROM_KEIL_sct.jpg)


## 4. Build & image generation

### Keil targets recommended

| Target | Output |
|-----|------|
| LDROM_BOOT | `LDROM_Bootloader.bin` |
| APROM_BOOT_EXT | `APROM_Bootloader.bin` @ `0x1E000` |

### Scatter file for boot code

```c
LOAD_ROM_1  0x100000 0x1000
{
	LDROM_Bootloader.bin  0x100000 0x1000
	{
		startup_m2a23.o (RESET, +FIRST)
        .ANY (+RO)
	}
	
	SRAM  0x20000000 0x6000
	{
		* (+RW, +ZI)
	}
}

LOAD_ROM_2  0x1E000 0x2000
{
	APROM_Bootloader.bin  0x1E000 0x2000
	{
        .ANY (+RO)
	}
}
```

### Checksum strategy actual project

* CRC32 over: `0x0000_0000 ~ 0x0001_DFFB`
* Stored at: `0x0001_DFFC`
* Boot compares SW CRC32 vs stored value

---

<a id="article_tools"></a>

## 5. Tool settings

### ICP tool mandatory first step

* Program:
  * `LDROM_Bootloader.bin` → LDROM
  * `APROM_Bootloader.bin` → APROM @ `0x1E000`

![](img/LDROM_ICP_Update.jpg)

* CONFIG:
  * **Boot from LDROM WITH IAP**

![](img/LDROM_ICP_Config.jpg)

### ISP tool UART

* **UART COM PORT**
* Select **APROM** and **Reset & Run**
* During update:
  * Progress bar shown on UART1
  * ISP protocol on UART0 only


![](img/boot_from_LDROM_to_APROM.jpg)

---

<a id="article_log"></a>

## 6. UART log & progress bar

```c

#define LDROM_DEBUG(format, args...) 		printf("\033[1;36m" "[LDROM]" format "\033[0m", ##args)

```

Progress bar width=10:

```
[LDROM] [#####-----] 50%
```

* Printed **after WriteData only**

---

<a id="article_summary"></a>

## 7. Summary training takeaway

* Boot is **policy-driven** `isp_config.c`
* Application controls update entry by **checksum invalidation**

![](img/LDROM_checksum_err.jpg)

* Dual UART avoids ISP/log interference
* CRC32 is the single source of truth for boot decision

---

# Appendix: Extended Build / Tool Details

<a id="appendix_top"></a>

# Agenda

* Boot code in LDROM,APROM image generation
* ISP tool operation
* SRecord post-build merge + CRC32

---

<a id="article_split_binary"></a>

# Boot code: split into 2 binaries LDROM + APROM end @ 0x1E000

## Why split?

* **LDROM size is limited** M2A23 LDROM is 4 KB, but training bootloader often needs:
  * protocol + CRC32 + log + timeouts + safety checks

## layout default

* `APP`: `0x0000_0000` ~ `0x0001_DFFF`  0x1E000 bytes
* `LDROM ext in APROM`: `0x0001_E000` ~ `APROM_END`  boot extension, fixed address
* `LDROM`: device LDROM region 4 KB

## output artifacts

* `LDROM_Bootloader.bin` boot code stage-1
* `APROM_Bootloader.bin` boot code stage-2, linked at APROM@0x1E000
* `APROM_application.bin` app code linked at 0x0000_0000, size ≤ 0x1E000, includes CRC word

![](img/APROM_KEIL_output_file.jpg)


[back to top](#article_top)

---

<a id="article_isp_tool"></a>

# ISP tool settings UART ISP update

## Typical steps UART

Connect ISP UART UART0 (target PCB) to PC USB-to-UART (UART bride) , Open ISP tool (SW)
1. Select UART port & baud rate
2. Click “Connect” ( if MCU under boot mode , will stay with connected)
3. Load image:
   * APROM: load `APROM_application.bin`
4. Select `APROM`
5. Select `Reset and Run`
6. execute Program `Start`

![](img/ISP_connect.jpg)

7. under ISP code tool , during upgrade application code

![](img/ISP_during_update.jpg)

8. under boot code , during upgrade application code

![](img/LDROM_during_upgrade.jpg)

## Notes

* Bootloader may have a timeout window; connect sequence matters.
* After update, ensure CRC word is correct; otherwise boot will stay in ISP.

[back to top](#article_top)

---

<a id="article_srecord"></a>

# SRecord settings merge + CRC32 append

## Use cases

* Fill holes with 0xFF
* KEIL setting : after compile , generate checksum with by batch file
![](img/APROM_KEIL_checksum_calculate.jpg)
![](img/APROM_SRecord_cmd_file.jpg)

**generateChecksum.bat**

```c

@echo off
setlocal EnableDelayedExpansion

:: MODIFY checksum_config.cmd only
call checksum_config.cmd

:: DO NOT EDIT checksum_flow_gen.cmd
:: It is auto-generated every build
:: generate srec script with expanded values
(
echo obj\APROM_application.bin -binary
echo -crop %APP_START% %APP_CRC_END%
echo -fill 0xFF %APP_START% %APP_CRC_END%
echo -crc32-l-e %CRC_POS%
echo -crop %CRC_POS% %CRC_END%
) > checksum_flow_gen.cmd

:: dump checksum
srec_cat @checksum_flow_gen.cmd -Output - -HEX_Dump

:: update binary
srec_cat @checksum_flow_gen.cmd ^
    obj\APROM_application.bin -binary ^
    -fill 0xFF %APP_START% %APP_CRC_END% ^
    -Output obj\APROM_application.bin -binary

:: generate hex
srec_cat obj\APROM_application.bin -binary ^
    -Output obj\APROM_application.hex -intel

```

**checksum_flow.cmd** ==(the only file need to modify)==
```c
:: ===== Application layout configuration =====

:: application start
set APP_START=0x0000

:: checksum calculate end (exclude checksum field)
set APP_CRC_END=0x1DFFC

:: checksum field start
set CRC_POS=0x1DFFC

:: checksum field size (CRC32 = 4 bytes)
set CRC_SIZE=0x0004

:: checksum field end
set CRC_END=0x1E000

```


**checksum_config.cmd**
```c
# input binary
obj\APROM_application.bin -binary

# keep code area for CRC calculation
-crop %APP_START% %APP_CRC_END%

# fill unused area with 0xFF
-fill 0xFF %APP_START% %APP_CRC_END%

# calculate CRC32 (little-endian) and place at CRC_POS
-crc32-l-e %CRC_POS%

# keep only checksum field
-crop %CRC_POS% %CRC_END%

```


[back to top](#article_top)

---