 
 ------------------------------------ 

# CoreI3C Bare Metal Driver
 
-----------------------------------------
## Table of Contents 
  - [Introduction](#introduction)

  - [Features (v2.0)](#features-(v2.0))

  - [Known Limitations (v2.0)](#known-limitations-(v2.0))

  - [Unsupported Features in Hardware (planned for future releases)](#unsupported-features-in-hardware-(planned-for-future-releases))

  - [Driver Configuration](#driver-configuration)

  - [Theory of Operation](#theory-of-operation)

  - [Initialization](#initialization)

  - [Dynamic Address Assignment](#dynamic-address-assignment)

  - [Private Transfers](#private-transfers)

  - [Common Command Codes (CCC)](#common-command-codes-(ccc))

  - [Interrupt Management](#interrupt-management)

  - [Bus Control](#bus-control)

- [Types](#types)
  - [i3c_status_t](#i3cstatust)
  - [i3c_dev_info_t](#i3cdevinfot)
  - [i3c_address_t](#i3caddresst)
  - [i3c_cfg_t](#i3ccfgt)
  - [i3c_instance_t](#i3cinstancet)

- [Constants](#constants)

- [Functions](#functions)
  - [I3C_init_controller](#i3cinitcontroller)
  - [I3C_do_daa](#i3cdodaa)
  - [I3C_abort](#i3cabort)
  - [I3C_resume](#i3cresume)
  - [I3C_write_ccc](#i3cwriteccc)
  - [I3C_read_ccc](#i3creadccc)
  - [I3C_write](#i3cwrite)
  - [I3C_read](#i3cread)
  - [I3C_write_read](#i3cwriteread)
  - [I3C_read_response](#i3creadresponse)
  - [I3C_get_status](#i3cgetstatus)
  - [I3C_enable_irq](#i3cenableirq)
  - [I3C_disable_irq](#i3cdisableirq)
  - [I3C_clear_irq](#i3cclearirq)
  - [I3C_get_irq_status](#i3cgetirqstatus)
  - [I3C_set_dat_entry](#i3csetdatentry)
  - [I3C_transfer_complete](#i3ctransfercomplete)
  - [I3C_get_device_info](#i3cgetdeviceinfo)
  - [I3C_wait_complete](#i3cwaitcomplete)

<div id="TitlePage" data-type="text">

# Introduction
The CoreI3C IP implements the MIPI I3C Basic specification v1.1 in controller
mode, providing an AXI4 host interface for FPGA-based SoC designs. This driver
provides a bare-metal API for all supported CoreI3C v2.0 features.

This driver provides a set of functions for managing CoreI3C in a 
bare-metal system, where no operating system is available. This driver can be
adapted to be used as a part of an operating system, but the implementation of
the adaptation layer between the driver and the operating system's driver model
is outside the scope of this driver.

# Features (v2.0)
The following is the list of CoreI3C driver features: 

  - I3C Controller mode (SDR) up to 12.5 Mbps push-pull clock
  - Dynamic Address Assignment (DAA) using ENTDAA and SETDASA procedures
  - Legacy I2C device coexistence on the same bus
  - Broadcast and Direct Common Command Codes (CCC) with optional defining byte
  - Private write transfers to target devices
  - Private read transfers from target devices
  - Write-then-read (combined) transfers for register access patterns 
  - Full interrupt subsystem: Enable, Disable, Clear, and Status read 
  - Response descriptor parsing with transaction ID correlation 
  - Device Address Table (DAT) and Device Characteristic Table (DCT) management 
  - Transfer abort and bus resume after error halt
  - Polling and interrupt-driven operation models

# Known Limitations (v2.0)
IBI (In-Band Interrupt): The hardware provides IBI reception capability, but
this driver does not implement IBI handling.

The current CoreI3C v2.0 hardware supports only SDR0 mode (up to 12.5 MHz).
It does not support extended SDR speeds (SDR1 through SDR4) or speed selection 
for individual devices. On a mixed-speed bus, the only
workaround would be to reprogram the Push-Pull timing register before every
transfer to a different-speed device, which is not practical and is not
implemented in this driver. When a future hardware releases include support for 
the per-command field, the driver will be updated to allow the caller to select the
transfer speed on a per-device or per-transfer basis without any register
reprogramming.

Bus Condition Control (START/STOP/Repeated-START): In the current CoreI3C
v2.0 hardware, START and STOP conditions are generated exclusively by the
hardware state machine based on the command descriptor. The software has no control
over the issuance of START or STOP; these are entirely managed by the
controller protocol engine. In future hardware revisions, if exposes explicit 
bus, the driver will be updated to provide software-controlled START/STOP 
generation for each transfer.

Additionally, Repeated-START (Sr) is not supported in this release. The hardware
cannot generate a Repeated-START condition between transfers. The
I3C_write_read() function currently implements the write-then-read pattern as
two discrete transfers (write followed by a separate read) rather than a single
combined transfer with an Sr between the write and read phases. Repeated-START
support will be added in a future hardware release.

Command Descriptor Constraints: Only CMD_ATTR values 0x0 (Regular Data Transfer)
and 0x2 (Address Assignment) are implemented. Submitting a command with any
other CMD_ATTR value (0x1, 0x3–0x7) produces a response descriptor with
ERR_STATUS = 0xA (NOT_SUPPORTED). The hardware does not execute such commands.

# Unsupported Features in Hardware (planned for future releases)
  - HDR modes (HDR-DDR, HDR-TSP, HDR-TSL)
  - Hot-Join
  - Secondary Controller
  - Target mode
  - Multi-Lane data transfer

# Driver Configuration
Call I3C_init_controller() once per CoreI3C hardware instance before any other
driver function. The caller must provide a statically allocated i3c_instance_t
and a populated i3c_cfg_t configuration structure.

Example:


```
    #include "core_i3c.h"

    static i3c_instance_t g_i3c;

    int main(void)
    {
        i3c_address_t addr_list[3];
        uint8_t       tx[4] = {0xA0u, 0xB1u, 0xC2u, 0xD3u};

        i3c_cfg_t cfg = {
            .aclk_freq_hz    = 50000000u,
            .scl_freq_hz     = 12500000u,
             .i2c_dev_present = 0u
         };

         I3C_init_controller(&g_i3c, 0x44000000u, 0x08u, &cfg);

        return 0;
    }
```
# Theory of Operation
The CoreI3C software driver is designed to control multiple instances of CoreI3C
IP. Each CoreI3C instance in the hardware design is associated with a single
instance of the i3c_instance_t structure in the software. The contents of these
data structures are initialized by calling the I3C_init_controller() function. A
pointer to the structure is passed to the subsequent driver functions to
identify the CoreI3C hardware instance on which the operation should be
performed.

  - Initialization
  - Dynamic Address Assignment
  - Private Transfers
  - Common Command Codes (CCC)
  - Interrupt Management
  - Bus Control

# Initialization
The CoreI3C driver must be initialized before performing any bus operations. The
I3C_init_controller() function performs a soft-reset of the core, resets all
internal FIFOs, calculates and programs the SCL Open-Drain and Push-Pull timing
registers from the provided ACLK and SCL frequencies. It sets the controller's own
dynamic address, and also enables the I3C bus. The caller must provide a populated
i3c_cfg_t structure specifying the system clock frequency (aclk_freq_hz), the
desired I3C SCL frequency (scl_freq_hz) and whether legacy I2C devices are
present on the bus (i2c_dev_present).

# Dynamic Address Assignment
After initialization, the bus topology must be established through Dynamic
Address Assignment (DAA). The I3C_do_daa() function handles the full DAA
procedure. It first broadcasts RSTDAA to clear existing assignments, then
processes the user-provided address list sequentially—registering I2C devices
in the DAT, issuing SETDASA CCCs for devices with known static addresses, and
finally issuing ENTDAA to discover and assign dynamic addresses to all remaining
unaddressed I3C targets. After I3C_do_daa() returns, this_i3c->dev_count
contains the number of assigned devices and this_i3c->char_table[] holds each
device's PID, BCR, DCR and assigned address.

# Private Transfers
Once addresses are assigned, data can be exchanged with targets using private
transfers. I3C_write() function initiates a controller write transfer, placing data into
the Write FIFO and submitting a command descriptor. I3C_read() function initiates a
controller read transfer, submitting a read command and draining the Read FIFO
into the caller's buffer upon completion. I3C_write_read() function combines both
operations — it writes an offset/register address, then issues a repeated-START
and reads back the response. This is the standard pattern for reading registers
from I3C target devices.

# Common Command Codes (CCC)
The I3C protocol defines Common Command Codes for bus management. The driver
provides two functions: I3C_write_ccc() for write-type CCCs (for both broadcast and direct) and
I3C_read_ccc() for read-type direct CCCs. CCC bytes from the range 0x00-0x7F are
broadcast commands that address all targets simultaneously (for example, RSTDAA, ENEC and 
DISEC). CCC bytes from the range 0x80-0xFE are direct commands that address a
single target (for example, GETBCR, GETPID and SETMWL). Some CCCs require a defining byte —
set the I3C_OPT_DEF_BYTE option flag and provide the byte value using the
defining_byte parameter.

# Interrupt Management
The CoreI3C hardware provides a two-level interrupt system: status logging, 
which is always active after initialization and CPU signal enable, which is optional. Status logging
records all events into the INTR_STAT and CMND_RSPNS_INTR_STAT registers, 
regardless of whether signaling is enabled. This allows polling-based operation
using I3C_get_irq_status() at any time. 

For interrupt-driven operation, call I3C_enable_irq() with the desired interrupt
mask to route events to the INT_O output pin. Inside the ISR (Interrupt Service Routine), call
I3C_get_irq_status() to read pending events, process them and then call
I3C_clear_irq() to acknowledge. Use I3C_transfer_complete() in the ISR to
complete pending read transfers. I3C_disable_irq() disables CPU signaling for
selected sources without affecting status logging.

# Bus Control
I3C_abort() terminates the ongoing transfer at the next byte
boundary. I3C_resume() resumes bus operation after the controller enters the
halt state due to a transfer error. I3C_get_status() reads the PRESENT_STATE
register to determine whether this instance is the active controller.

</div>


# Types

 ---------------- 
<a name="i3cstatust"></a>
## i3c_status_t
<a name="prototype"></a>
### Prototype 

<div id="Types$i3c_status_t$prototype" data-type="code">

 ``` 
    typedef enum {
        I3C_SUCCESS = 0, <--Operation completed successfully  

        I3C_IN_PROGRESS, <--Transfer submitted, awaiting completion 
        I3C_ERR_PARAM, <--Invalid parameter  

        I3C_ERR_FIFO_FULL, <--Target FIFO is full; cannot queue data  

        I3C_ERR_FIFO_EMPTY, <--Source FIFO is empty; no data available 
        I3C_ERR_NOT_CTRL, <--Not the active controller on the bus  

        I3C_ERR_NACK, <--Target responded with NACK  

        I3C_ERR_TIMEOUT, <--Command sequence timeout  

        I3C_ERR_ABORTED, <--Transfer was aborted  

        I3C_ERR_INTERNAL, <--Non-recoverable internal core error  

        I3C_ERR_BUS_BUSY, <--Bus is currently busy  

        I3C_ERR_DAA_FAILED <--DAA procedure failed (no devices found) 
    } i3c_status_t;


 ``` 


</div>

<a name="description"></a>
### Description 

<div id="Types$i3c_status_t$description" data-type="text">

Return status codes used by all CoreI3C driver functions.

| Constant | Value | Description |
| ----- | ----- | ----- |
| I3C_SUCCESS | 0 | Operation completed successfully |
| I3C_IN_PROGRESS | 1 | Transfer submitted, awaiting completion |
| I3C_ERR_PARAM | 2 | Invalid parameter |
| I3C_ERR_FIFO_FULL | 3 | Target FIFO is full; cannot queue data |
| I3C_ERR_FIFO_EMPTY | 4 | Source FIFO is empty; no data available |
| I3C_ERR_NOT_CTRL | 5 | Not the active controller on the bus |
| I3C_ERR_NACK | 6 | Target responded with NACK |
| I3C_ERR_TIMEOUT | 7 | Command sequence timeout |
| I3C_ERR_ABORTED | 8 | Transfer was aborted |
| I3C_ERR_INTERNAL | 9 | Non-recoverable internal core error |
| I3C_ERR_BUS_BUSY | 10 | Bus is currently busy |
| I3C_ERR_DAA_FAILED | 11 | DAA procedure failed (no devices found) |


</div>


 --------------------------- 
<a name="i3cdevinfot"></a>
## i3c_dev_info_t
<a name="prototype"></a>
### Prototype 

<div id="Types$i3c_dev_info_t$prototype" data-type="code">

``` 
    typedef struct i3c_dev_info {
        uint16_t vendor_id; <--15-bit manufacturer ID (PID bits [47:33]) 
        uint16_t part_id; <--16-bit part number (PID bits [31:16])  

        uint16_t instance_id; <--4-bit instance ID (PID bits [15:12])  

        uint16_t additional_info; <--12-bit extra info (PID bits [11:0])  

        uint8_t id_type; <--0 = vendor fixed ID, 1 = random ID  

        uint8_t bcr; <--Bus Characteristics Register  

        uint8_t dcr; <--Device Characteristics Register  

        uint8_t address; <--Assigned dynamic or static address  

    } i3c_dev_info_t ;
  ``` 


</div>

<a name="description"></a>
### Description 

<div id="Types$i3c_dev_info_t$description" data-type="text">

The i3c_dev_info_t structure holds the device characteristics for one target.
For SETI2C and SETDASA devices, the user pre-fills this before calling
I3C_do_daa(). During the DAA process, the driver copies this information into the char_table[]. For ENTDAA 
devices, this structure is not used in the address list. Hardware discovered
data (PID/BCR/DCR/address) is written directly into this_i3c->char_table[] by
the driver after ENTDAA process completes. To identify a specific device after DAA, pre-
fill a separate named i3c_dev_info_t with the known PID/BCR/DCR and search
this_i3c->char_table[] by matching fields.

The 48-bit Provisional ID is broken down into its constituent fields.

| Field | Description |
| ----- | ----- |
| vendor_id | 15-bit manufacturer ID from PID [47:33] |
| part_id | 16-bit part number from PID [31:16] |
| instance_id | 4-bit instance ID from PID [15:12] |
| additional_info | 12-bit extra info from PID [11:0] |
| id_type | PID[32]: 0 = vendor fixed ID, 1 = random ID |
| bcr | Bus Characteristics Register |
| dcr | Device Characteristics Register |
| address | Assigned/active address for this device |


</div>


 --------------------------- 
<a name="i3caddresst"></a>
## i3c_address_t
<a name="prototype"></a>
### Prototype 

<div id="Types$i3c_address_t$prototype" data-type="code">

``` 
    typedef struct i3c_address {
        i3c_dev_info_t *dev_info; <--Pointer to device info  

        uint8_t address; <--Assigned dynamic address  

        uint8_t static_address; <--Static/I2C address (user pre-fills) 
        uint8_t flags; <--I3C_ADDR_FLAG_xxx  

    } i3c_address_t ;
  ``` 


</div>

<a name="description"></a>
### Description 

<div id="Types$i3c_address_t$description" data-type="text">

The i3c_address_t structure represents a single device slot in the address list
passed to I3C_do_daa(). Before calling I3C_do_daa(), the user must pre-fill the 
entire list. The list must be terminated by an entry with flags = I3C_ADDR_FLAG_EOL.

| Field | Description |
| ----- | ----- |
| dev_info | Pointer to the device information. Before calling I3C_do_daa(), user must pre-fills the SETI2C/SETDASA entries. It must be set to NULL for ENTDAA entries — the driver never reads or writes this field for ENTDAA. |
| address | Assigned dynamic address (filled by driver for ENTDAA/SETDASA; same as static_address for I2C) |
| static_address | Static address (pre-filled by user for I2C/SETDASA) |
| flags | One of I3C_ADDR_FLAG_SETI2C / _SETDASA / _ENTDAA / _EOL |

Processing order inside I3C_do_daa():

 1. SETI2C entries — DAT populated with static addr + I2C flag (no bus)
 2. SETDASA entries — Each gets a SETDASA CCC on the bus (one by one)
 3. ENTDAA entries — All participate in a single ENTDAA command


</div>


 --------------------------- 
<a name="i3ccfgt"></a>
## i3c_cfg_t
<a name="prototype"></a>
### Prototype 

<div id="Types$i3c_cfg_t$prototype" data-type="code">

``` 
    typedef struct i3c_cfg {
        uint32_t aclk_freq_hz; <--AXI4 system clock frequency in Hz  

        uint32_t scl_freq_hz; <--Target I3C SCL bus frequency in Hz  

        uint8_t i2c_dev_present; <--1 = legacy I2C devices present on bus  

    } i3c_cfg_t ;
  ``` 


</div>

<a name="description"></a>
### Description 

<div id="Types$i3c_cfg_t$description" data-type="text">

The i3c_cfg_t structure holds all configuration parameters passed to
I3C_init_controller(). The caller must populate this structure before calling
init.

| Field | Description |
| ----- | ----- |
| aclk_freq_hz | AXI4 system clock frequency in Hz (for example, 100000000) |
| scl_freq_hz | Target I3C SCL frequency in Hz (for example, 12500000) |
| i2c_dev_present | 1 if legacy I2C devices are present on the bus |

The driver internally calculates the OD and PP SCL timing register values using 
aclk_freq_hz and scl_freq_hz. See "SCL Timing Calculation" section above for the
formula and constraints.

Example for 100 MHz ACLK targeting 12.5 MHz I3C bus:


```
    i3c_cfg_t cfg = {
        .aclk_freq_hz    = 100000000u,  // 100 MHz ACLK
        .scl_freq_hz     = 12500000u,   // 12.5 MHz I3C bus (SDR0)
        .i2c_dev_present = 0u
    };
```

</div>


 --------------------------- 
<a name="i3cinstancet"></a>
## i3c_instance_t
<a name="prototype"></a>
### Prototype 

<div id="Types$i3c_instance_t$prototype" data-type="code">

``` 
    typedef struct i3c_instance {
        addr_t base_addr; 
        uint8_t tid; 
        uint16_t dat_offset; 
        uint8_t dat_size; 
        uint16_t dct_offset; 
        uint8_t dct_size; 
        uint8_t dev_count; 
        uint8_t non_entdaa; 
        uint8_t addr_list_count; 
        volatile i3c_status_t controller_status; 
        i3c_status_t last_status; 
        const uint8_t *controller_tx_buffer; 
        uint16_t controller_tx_size; 
        uint16_t controller_tx_idx; 
        uint8_t *controller_rx_buffer; 
        uint16_t controller_rx_size; 
        uint32_t *controller_rx_len; 
        i3c_dev_info_t char_table; 
        uint8_t address_flags; 
    } i3c_instance_t ;
  ``` 


</div>

<a name="description"></a>
### Description 

<div id="Types$i3c_instance_t$description" data-type="text">

The i3c_instance_t structure identifies a CoreI3C hardware instance.
The I3C_init_controller() function initializes this structure. You must provide 
a pointer to an initialized instance as the first parameter to all driver 
functions.

The instance stores internal per-device state including the device
characteristic table, address flags and the pending transfer context for interrupt-
driven operation.


</div>


 --------------------------- 

# Constants

 ---------------- 
<div id="Constants" data-type="text">

<a name="transfer-option-flags"></a>
### Transfer Option Flags

Transfer option flags used in I3C_write(), I3C_read(), I3C_write_read(),
I3C_write_ccc() and I3C_read_ccc().

| Constant | Description |
| ----- | ----- |
| I3C_OPT_NONE | No options — normal transfer |
| I3C_OPT_INT_SAFE | No need to enable/disable interrupts — caller manages completion directly. Does not arm transfer interrupt sources (CMND_RDY, WRT, RD). Used internally by I3C_do_daa() for blocking RSTDAA. |
| I3C_OPT_DEF_BYTE | CCC has a Defining Byte — sets DBP bit in descriptor. The defining byte value is passed using the defining_byte parameter in I3C_write_ccc() and I3C_read_ccc(). |

### Address List Flags

| Constant | Description |
| ----- | ----- |
| I3C_ADDR_FLAG_SETI2C | Legacy I2C device (static addr, no bus) |
| I3C_ADDR_FLAG_SETDASA | I3C device with static addr (SETDASA) |
| I3C_ADDR_FLAG_ENTDAA | I3C device needing dynamic assignment |
| I3C_ADDR_FLAG_EOL | End of list sentinel |

### Command/Response Interrupt Sources

| Constant | Bit | Description |
| ----- | ----- | ----- |
| I3C_CR_IRQ_TRANSFR_ERR | 9 | Transfer error on the I3C bus |
| I3C_CR_IRQ_TRANSFR_ABRT | 5 | Transfer aborted |
| I3C_CR_IRQ_RSPNS_RDY | 4 | Response descriptor ready in STATUS FIFO |
| I3C_CR_IRQ_CMND_RDY | 3 | Command transfer completed |
| I3C_CR_IRQ_IBI_RECV | 2 | IBI received (IBI FIFO has at least one entry) |
| I3C_CR_IRQ_WRT | 1 | Write FIFO has at least one data entry |
| I3C_CR_IRQ_RD | 0 | Read FIFO has at least one data entry |

### General Interrupt Sources (from corei3c_regs.h)

| Constant | Bit | Description |
| ----- | ----- | ----- |
| COREI3C_I3C_INTR_STAT_CMD_STALL_MASK | 12 | Command sequence stall (warn) |
| COREI3C_I3C_INTR_STAT_SEQ_CANCEL_MASK | 11 | Cancelled transaction sequence |
| COREI3C_I3C_INTR_STAT_INTERNAL_ERR_MASK | 10 | Non-recoverable internal error |

### CCC Range Boundaries

| Constant | Value | Description |
| ----- | ----- | ----- |
| I3C_CCC_BROADCAST_MAX | 0x7F | Highest broadcast CCC value |
| I3C_CCC_DIRECT_MIN | 0x80 | Lowest direct CCC value |

### OD Timing Constraints

| Constant | Value | Description |
| ----- | ----- | ----- |
| I3C_OD_THIGH_MAX_NS | 41 | Maximum OD high time in ns |
| I3C_OD_TLOW_MIN_NS | 200 | Minimum OD low time in ns |

</div>

# Functions

 ---------------- 
<a name="i3cinitcontroller"></a>
## I3C_init_controller
<a name="prototype"></a>
### Prototype 

<div id="Functions$I3C_init_controller$prototype" data-type="code">

    i3c_status_t
    I3C_init_controller
    (
        i3c_instance_t * this_i3c,
        addr_t base_addr,
        uint8_t ser_address,
        const i3c_cfg_t * cfg
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$I3C_init_controller$description" data-type="text">

The I3C_init_controller() function initializes the driver. It sets the base
address for the CoreI3C instance, initializes the i3c_instance_t data structure,
applies the configuration provided in the cfg parameter including SCL timing and
operating settings and enables the I3C bus. This function must be called before
calling any other CoreI3C driver functions.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_i3c
<div id="Functions$I3C_init_controller$description$parameters$this_i3c" data-type="text" data-name="this_i3c">

The this_i3c parameter is a pointer to an i3c_instance_t structure that holds
all data related to the CoreI3C instance being initialized. A pointer to this
data structure is used in all subsequent calls to the CoreI3C driver functions
that operate on this CoreI3C instance.


</div>

#### base_addr
<div id="Functions$I3C_init_controller$description$parameters$base_addr" data-type="text" data-name="base_addr">

The base_addr parameter is the base address in the processor's memory map for
the registers of the CoreI3C instance being initialized.


</div>

#### ser_address
<div id="Functions$I3C_init_controller$description$parameters$ser_address" data-type="text" data-name="ser_address">

The ser_address parameter is the 7-bit dynamic address, which the controller assigns to
itself as the active primary controller. This address is written to the
CNTRL_DEVICE_ADDR register.


</div>

#### cfg
<div id="Functions$I3C_init_controller$description$parameters$cfg" data-type="text" data-name="cfg">

The cfg parameter is a pointer to an i3c_cfg_t structure containing the
configuration to apply. It must not be NULL. The caller must set cfg->aclk_freq_hz
to the AXI4 system clock frequency in Hz and cfg->scl_freq_hz to the target I3C
SCL bus frequency in Hz. The driver calculates OD and PP timing register values
internally.


</div>

<a name="return"></a>
### Return
<div id="Functions$I3C_init_controller$description$return" data-type="text">

| Constants | Description |
| ----- | ----- |
| I3C_SUCCESS | Successful initialization |
| I3C_ERR_PARAM | this_i3c or cfg is NULL |
| I3C_ERR_TIMEOUT | Soft reset or FIFO reset did not complete within expected time |


</div>

##### Example
<div id="Functions$I3C_init_controller$description$example$Example 1" data-type="text" data-name="Example">

</div>

<div id="Functions$I3C_init_controller$description$example$Example 1" data-type="code" data-name="Example">


```
    static i3c_instance_t g_i3c;

    i3c_cfg_t cfg = {
        .aclk_freq_hz    = 100000000u,
        .scl_freq_hz     = 12500000u,
        .i2c_dev_present = 0u
    };

    I3C_init_controller(&g_i3c, 0x44000000u, 0x08u, &cfg);
```

</div>


 -------------------------------- 
<a name="i3cdodaa"></a>
## I3C_do_daa
<a name="prototype"></a>
### Prototype 

<div id="Functions$I3C_do_daa$prototype" data-type="code">

    void
    I3C_do_daa
    (
        i3c_instance_t * this_i3c,
        i3c_address_t * addr_list,
        uint32_t options
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$I3C_do_daa$description" data-type="text">

The I3C_do_daa() function performs Dynamic Address Assignment on the I3C bus. It
processes the address list sequentially based on each entry's flags:

 1. SETI2C entries — Writes static address into DAT with the I2C flag (no bus).
 2. SETDASA entries — Issues a SETDASA CCC per device to assign a known dynamic 
address to a device that has a static address.
 3. ENTDAA entries — Issues ENTDAA to discover and assign dynamic addresses to 
all remaining unaddressed I3C devices on the bus.

When options = 0 (fresh DAA), the function first issues RSTDAA to clear all
existing dynamic address assignments, then processes the full list.

When options != 0 (continuation), the function skips the RSTDAA and SETDASA phases.
The addr_list parameter is not re-scanned — the remaining device count is
derived from the instance's cached dev_count and addr_list_count set during the
original options=0 call. Only ENTDAA is issued for the remaining unassigned DAT
slots. This is used after a hot-join event to assign new devices without
disturbing already-assigned devices.

After completion, the discovered device information (PID, BCR, DCR and address) is stored
in the instance's internal char_table[]. The total device count is available using
this_i3c->dev_count.

For SETDASA entries: Set static_address (device's static addr) and address
(desired dynamic addr to assign). Set dev_info to point to the user-provided device
characteristics.

For ENTDAA entries: Set address to the desired dynamic address to assign. Set
dev_info to NULL — the driver never reads or writes this field for ENTDAA
entries. After the DAA process completes, hardware-discovered data (PID/BCR/DCR and assigned
address) is available in this_i3c->char_table[]. To identify a specific known
device, pre-fill a separate named i3c_dev_info_t with its expected PID/BCR/DCR
before calling I3C_do_daa(). Then, search this_i3c->char_table[] by matching those
fields after DAA returns.

Any static I2C assignments (SETI2C) or addresses assigned using SETDASA must be
placed first in the array before any dynamically assigned addresses set using ENTDAA.
Placing SETI2C or SETDASA entries after any ENTDAA entry will cause those
SETI2C/SETDASA entries to be silently skipped with no error returned.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_i3c
<div id="Functions$I3C_do_daa$description$parameters$this_i3c" data-type="text" data-name="this_i3c">

The this_i3c parameter is a pointer to an i3c_instance_t structure that holds
all data related to the CoreI3C instance being initialized. A pointer to this
data structure is used in all subsequent calls to the CoreI3C driver functions
that operate on this CoreI3C instance.


</div>

#### addr_list
<div id="Functions$I3C_do_daa$description$parameters$addr_list" data-type="text" data-name="addr_list">

After completion, the discovered device information (PID, BCR, DCR, address) is stored
in the instance's internal char_table[]. The total device count is available using
this_i3c->dev_count.

The addr_list parameter is a pointer to a pre-allocated array of i3c_address_t
structures describing the expected bus topology. The array must be terminated by
an entry with flags = I3C_ADDR_FLAG_EOL.

For SETI2C entries: Set static_address and dev_info (user pre-fills BCR/DCR if known, or zeroes).

For SETDASA entries: Set static_address (device's static addr) and address
(desired dynamic addr to assign). Set dev_info pointing to user-provided device
characteristics.

For ENTDAA entries: Set address to the desired dynamic address to assign. Set
dev_info to NULL — the driver never reads or writes this field for ENTDAA
entries. After the DAA process completes, the hardware-discovered data (PID/BCR/DCR and assigned
address) is available in this_i3c->char_table[]. To identify a specific known
device, pre-fill a separate named i3c_dev_info_t with its expected PID/BCR/DCR
before calling I3C_do_daa() and search this_i3c->char_table[] by matching those
fields after the DAA process returns.

Any static I2C assignments (SETI2C) or addresses assigned through SETDASA must be
placed first in the array before any dynamically assigned addresses set using ENTDAA.
Placing SETI2C or SETDASA entries after any ENTDAA entry will cause those
SETI2C/SETDASA entries to be silently skipped with no error returned.


</div>

#### options
<div id="Functions$I3C_do_daa$description$parameters$options" data-type="text" data-name="options">

The options parameter controls the DAA operation mode. Pass 0 for a fresh DAA
(RSTDAA + full list processing). Pass non-zero for continuation (ENTDAA only for
remaining devices).


</div>

<a name="return"></a>
### Return
<div id="Functions$I3C_do_daa$description$return" data-type="text">

This function does not return any value.


</div>

##### Example 1
<div id="Functions$I3C_do_daa$description$example$Example 1" data-type="text" data-name="Example 1">
ENTDAA-only bus (all devices discovered dynamically).


</div>

<div id="Functions$I3C_do_daa$description$example$Example 1" data-type="code" data-name="Example 1">


```
    #define MAX_DEVS  2u

    static i3c_address_t  address_list[MAX_DEVS + 1u];

    // All slots are I3C ENTDAA devices
    address_list[0].dev_info       = (i3c_dev_info_t *)0;
    address_list[0].address        = 0x09u;
    address_list[0].static_address = 0u;
    address_list[0].flags          = I3C_ADDR_FLAG_ENTDAA;

    address_list[1].dev_info       = (i3c_dev_info_t *)0;
    address_list[1].address        = 0x0Au;
    address_list[1].static_address = 0u;
    address_list[1].flags          = I3C_ADDR_FLAG_ENTDAA;

    address_list[2].flags          = I3C_ADDR_FLAG_EOL;

    I3C_do_daa(&g_i3c, &address_list[0], 0u);

    // After DAA, hardware-discovered data is in g_i3c.char_table[].
    // To find a specific device, search char_table[] by PID fields.
    // Example: find a device with vendor_id=0x0251, part_id=0x0000
    uint8_t found_addr = 0u;
    uint32_t i;
    for( i = 0u; i < (uint32_t)g_i3c.dev_count; i++ )
    {
        if( (g_i3c.char_table[i].vendor_id == 0x0251u) &&
            (g_i3c.char_table[i].part_id   == 0x0000u) )
        {
            found_addr = g_i3c.char_table[i].address;
            break;
        }
    }
```

</div>

##### Example 2
<div id="Functions$I3C_do_daa$description$example$Example 2" data-type="text" data-name="Example 2">
Mixed bus: 1 I2C at 0x50, 1 SETDASA at 0x30->0x0B, 2 ENTDAA.


</div>

<div id="Functions$I3C_do_daa$description$example$Example 2" data-type="code" data-name="Example 2">


```
    // Mixed bus: 1 I2C at 0x50, 1 SETDASA at 0x30->0x0B, 2 ENTDAA
    i3c_dev_info_t i2c_dev_info;
    i3c_dev_info_t setdasa_dev_info;
    i3c_address_t  address_list[5];

    // I2C device: user pre-fills dev_info with known characteristics
    address_list[0].dev_info       = &i2c_dev_info;
    address_list[0].address        = 0u;
    address_list[0].static_address = 0x50u;
    address_list[0].flags          = I3C_ADDR_FLAG_SETI2C;

    // SETDASA device: user pre-fills dev_info with known characteristics
    address_list[1].dev_info       = &setdasa_dev_info;
    address_list[1].address        = 0x0Bu;
    address_list[1].static_address = 0x30u;
    address_list[1].flags          = I3C_ADDR_FLAG_SETDASA;

    // ENTDAA devices: dev_info must be NULL — driver never reads or writes it
    address_list[2].dev_info       = (i3c_dev_info_t *)0;
    address_list[2].address        = 0x09u;
    address_list[2].static_address = 0u;
    address_list[2].flags          = I3C_ADDR_FLAG_ENTDAA;

    address_list[3].dev_info       = (i3c_dev_info_t *)0;
    address_list[3].address        = 0x0Au;
    address_list[3].static_address = 0u;
    address_list[3].flags          = I3C_ADDR_FLAG_ENTDAA;

    address_list[4].flags          = I3C_ADDR_FLAG_EOL;

    I3C_do_daa(&g_i3c, &address_list[0], 0u);
    // After DAA: ENTDAA device info is in g_i3c.char_table[2] and [3].
    // Search char_table[] by PID fields to identify specific devices.
```

</div>


 -------------------------------- 
<a name="i3cabort"></a>
## I3C_abort
<a name="prototype"></a>
### Prototype 

<div id="Functions$I3C_abort$prototype" data-type="code">

    i3c_status_t
    I3C_abort
    (
        i3c_instance_t * this_i3c
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$I3C_abort$description" data-type="text">

The I3C_abort() function aborts the ongoing transfer on the CoreI3C
bus. The CoreI3C controller terminates the transaction at the nearest
opportunity after the current data byte completes.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_i3c
<div id="Functions$I3C_abort$description$parameters$this_i3c" data-type="text" data-name="this_i3c">

The this_i3c parameter is a pointer to an i3c_instance_t structure that holds
all data related to the CoreI3C instance being initialized. A pointer to this
data structure is used in all subsequent calls to the CoreI3C driver functions
that operate on this CoreI3C instance.


</div>

<a name="return"></a>
### Return
<div id="Functions$I3C_abort$description$return" data-type="text">

This function returns I3C_SUCCESS.


</div>

##### Example
<div id="Functions$I3C_abort$description$example$Example 1" data-type="text" data-name="Example">

</div>

<div id="Functions$I3C_abort$description$example$Example 1" data-type="code" data-name="Example">


```
    I3C_abort(&g_i3c);
```

</div>


 -------------------------------- 
<a name="i3cresume"></a>
## I3C_resume
<a name="prototype"></a>
### Prototype 

<div id="Functions$I3C_resume$prototype" data-type="code">

    i3c_status_t
    I3C_resume
    (
        i3c_instance_t * this_i3c
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$I3C_resume$description" data-type="text">

The I3C_resume() function resumes CoreI3C bus operation after the controller has
entered the halt state. The CoreI3C enters the halt state when a transfer error
occurs. This function writes the RESUME bit to allow the next queued command to
proceed.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_i3c
<div id="Functions$I3C_resume$description$parameters$this_i3c" data-type="text" data-name="this_i3c">

The this_i3c parameter is a pointer to an i3c_instance_t structure that holds
all data related to the CoreI3C instance being initialized. A pointer to this
data structure is used in all subsequent calls to the CoreI3C driver functions
that operate on this CoreI3C instance.


</div>

<a name="return"></a>
### Return
<div id="Functions$I3C_resume$description$return" data-type="text">

This function returns I3C_SUCCESS.


</div>

##### Example
<div id="Functions$I3C_resume$description$example$Example 1" data-type="text" data-name="Example">

</div>

<div id="Functions$I3C_resume$description$example$Example 1" data-type="code" data-name="Example">


```
    // Inside an ISR or main loop after detecting CMD_STALL:
    I3C_resume(&g_i3c);
```

</div>


 -------------------------------- 
<a name="i3cwriteccc"></a>
## I3C_write_ccc
<a name="prototype"></a>
### Prototype 

<div id="Functions$I3C_write_ccc$prototype" data-type="code">

    i3c_status_t
    I3C_write_ccc
    (
        i3c_instance_t * this_i3c,
        uint8_t serial_addr,
        uint8_t ccc_byte,
        uint8_t defining_byte,
        const uint8_t * write_buffer,
        uint16_t write_size,
        uint32_t options
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$I3C_write_ccc$description" data-type="text">

The I3C_write_ccc() function issues a write-type Common Command Code (CCC) either to a
specific target device (direct CCC) or to all devices on the bus (broadcast
CCC). If serial_addr is I3C_BROADCAST_ADDR (0x7E), the command is issued as a
broadcast. Otherwise, it is issued as a direct write CCC to the specified target.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_i3c
<div id="Functions$I3C_write_ccc$description$parameters$this_i3c" data-type="text" data-name="this_i3c">

The this_i3c parameter is a pointer to an i3c_instance_t structure that holds
all data related to the CoreI3C instance being initialized. A pointer to this
data structure is used in all subsequent calls to the CoreI3C driver functions
that operate on this CoreI3C instance.


</div>

#### serial_addr
<div id="Functions$I3C_write_ccc$description$parameters$serial_addr" data-type="text" data-name="serial_addr">

The serial_addr parameter specifies the 7-bit dynamic address of the target for
a direct CCC, or I3C_BROADCAST_ADDR (0x7E) for a broadcast CCC.


</div>

#### ccc_byte
<div id="Functions$I3C_write_ccc$description$parameters$ccc_byte" data-type="text" data-name="ccc_byte">

The ccc_byte parameter specifies the CCC command byte to issue (e.g.
I3C_CCC_RSTDAA, I3C_CCC_SETMWL).


</div>

#### defining_byte
<div id="Functions$I3C_write_ccc$description$parameters$defining_byte" data-type="text" data-name="defining_byte">

The defining_byte parameter specifies the defining byte for CCCs that require
one (for example, ENEC, DISEC, SETMWL). Set to 0 if not applicable. When non-zero and
I3C_OPT_DEF_BYTE is set in options, the driver sets DBP=1 in DWORD0 and places
this value into DWORD1[7:0] of the command descriptor.


</div>

#### write_buffer
<div id="Functions$I3C_write_ccc$description$parameters$write_buffer" data-type="text" data-name="write_buffer">

The write_buffer parameter is a pointer to the data payload to transmit after
the CCC byte. It can be NULL if write_size is 0.


</div>

#### write_size
<div id="Functions$I3C_write_ccc$description$parameters$write_size" data-type="text" data-name="write_size">

The write_size parameter specifies the number of bytes in write_buffer.


</div>

#### options
<div id="Functions$I3C_write_ccc$description$parameters$options" data-type="text" data-name="options">

The options parameter specifies the transfer option flags. OR of
I3C_OPT_DEF_BYTE, I3C_OPT_INT_SAFE, and so on. Set I3C_OPT_DEF_BYTE when the CCC
requires a defining byte.


</div>

<a name="return"></a>
### Return
<div id="Functions$I3C_write_ccc$description$return" data-type="text">

| Constants | Description |
| ----- | ----- |
| I3C_SUCCESS | Successful command submission |
| I3C_ERR_FIFO_FULL | Command or Write FIFO is full |
| I3C_ERR_NOT_CTRL | This instance is not the active controller |


</div>

##### Example
<div id="Functions$I3C_write_ccc$description$example$Example 1" data-type="text" data-name="Example">

</div>

<div id="Functions$I3C_write_ccc$description$example$Example 1" data-type="code" data-name="Example">


```
    // Broadcast RSTDAA — no defining byte
    I3C_write_ccc(&g_i3c, I3C_BROADCAST_ADDR, I3C_CCC_RSTDAA,
                  0u, NULL, 0u, I3C_OPT_NONE);

    // Broadcast ENEC — enable IBI events (defining byte = 0x01)
    uint8_t enec_data = 0x01u;
    I3C_write_ccc(&g_i3c, I3C_BROADCAST_ADDR, I3C_CCC_ENEC,
                  0x01u, &enec_data, 1u,
                  I3C_OPT_DEF_BYTE | I3C_OPT_RESPONSE);
```

</div>


 -------------------------------- 
<a name="i3creadccc"></a>
## I3C_read_ccc
<a name="prototype"></a>
### Prototype 

<div id="Functions$I3C_read_ccc$prototype" data-type="code">

    i3c_status_t
    I3C_read_ccc
    (
        i3c_instance_t * this_i3c,
        uint8_t serial_addr,
        uint8_t ccc_byte,
        uint8_t defining_byte,
        uint8_t * read_buffer,
        uint16_t read_size,
        uint32_t * read_len,
        uint32_t options
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$I3C_read_ccc$description" data-type="text">

The I3C_read_ccc() function issues a read-type Direct CCC to a specific target
device and reads the response into read_buffer.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_i3c
<div id="Functions$I3C_read_ccc$description$parameters$this_i3c" data-type="text" data-name="this_i3c">

The this_i3c parameter is a pointer to an i3c_instance_t structure that holds
all data related to the CoreI3C instance being initialized. A pointer to this
data structure is used in all subsequent calls to the CoreI3C driver functions
that operate on this CoreI3C instance.


</div>

#### serial_addr
<div id="Functions$I3C_read_ccc$description$parameters$serial_addr" data-type="text" data-name="serial_addr">

The serial_addr parameter specifies the 7-bit dynamic address of the target
device.


</div>

#### ccc_byte
<div id="Functions$I3C_read_ccc$description$parameters$ccc_byte" data-type="text" data-name="ccc_byte">

The ccc_byte parameter specifies the CCC command byte to issue (e.g.
I3C_CCC_GETBCR, I3C_CCC_GETPID).


</div>

#### defining_byte
<div id="Functions$I3C_read_ccc$description$parameters$defining_byte" data-type="text" data-name="defining_byte">

The defining_byte parameter specifies the defining byte for CCCs that require
one (for example, GETMXDS). Set to 0 if not applicable.


</div>

#### read_buffer
<div id="Functions$I3C_read_ccc$description$parameters$read_buffer" data-type="text" data-name="read_buffer">

The read_buffer parameter is a pointer to a buffer to receive the CCC response
data.


</div>

#### read_size
<div id="Functions$I3C_read_ccc$description$parameters$read_size" data-type="text" data-name="read_size">

The read_size parameter specifies the maximum number of bytes to read into
read_buffer.


</div>

#### read_len
<div id="Functions$I3C_read_ccc$description$parameters$read_len" data-type="text" data-name="read_len">

The read_len parameter is a pointer to a uint32_t that receives the actual
number of bytes read. It must not be NULL.


</div>

#### options
<div id="Functions$I3C_read_ccc$description$parameters$options" data-type="text" data-name="options">

The options parameter specifies the transfer option flags.


</div>

<a name="return"></a>
### Return
<div id="Functions$I3C_read_ccc$description$return" data-type="text">

| Constants | Description |
| ----- | ----- |
| I3C_SUCCESS | Successful command submission |
| I3C_ERR_FIFO_FULL | Command FIFO is full |
| I3C_ERR_NOT_CTRL | This instance is not the active controller |


</div>

##### Example
<div id="Functions$I3C_read_ccc$description$example$Example 1" data-type="text" data-name="Example">

</div>

<div id="Functions$I3C_read_ccc$description$example$Example 1" data-type="code" data-name="Example">


```
    uint8_t bcr_val;
    uint32_t rx_len = 0u;
    I3C_read_ccc(&g_i3c, 0x12u, I3C_CCC_GETBCR, 0u,
                 &bcr_val, 1u, &rx_len, I3C_OPT_RESPONSE);
```

</div>


 -------------------------------- 
<a name="i3cwrite"></a>
## I3C_write
<a name="prototype"></a>
### Prototype 

<div id="Functions$I3C_write$prototype" data-type="code">

    i3c_status_t
    I3C_write
    (
        i3c_instance_t * this_i3c,
        uint8_t serial_addr,
        const uint8_t * write_buffer,
        uint16_t write_size,
        uint32_t options
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$I3C_write$description" data-type="text">

The I3C_write() function initiates a private write transaction for an I3C controller.
It writes data from write_buffer to the CoreI3C Write FIFO and submits a private
write command descriptor to the Command FIFO. The function returns after
initiating the transfer. The caller must not modify or free write_buffer until
the transfer is confirmed complete using the Status FIFO or the
I3C_CR_IRQ_CMND_RDY interrupt.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_i3c
<div id="Functions$I3C_write$description$parameters$this_i3c" data-type="text" data-name="this_i3c">

The this_i3c parameter is a pointer to an i3c_instance_t structure that holds
all data related to the CoreI3C instance being initialized. A pointer to this
data structure is used in all subsequent calls to the CoreI3C driver functions
that operate on this CoreI3C instance.


</div>

#### serial_addr
<div id="Functions$I3C_write$description$parameters$serial_addr" data-type="text" data-name="serial_addr">

The serial_addr parameter specifies the 7-bit dynamic address of the target
device.


</div>

#### write_buffer
<div id="Functions$I3C_write$description$parameters$write_buffer" data-type="text" data-name="write_buffer">

The write_buffer parameter is a pointer to the data that needs to be transmitted. The buffer must
remain valid until the transfer completes.


</div>

#### write_size
<div id="Functions$I3C_write$description$parameters$write_size" data-type="text" data-name="write_size">

The write_size parameter specifies the number of bytes to transmit from
write_buffer.


</div>

#### options
<div id="Functions$I3C_write$description$parameters$options" data-type="text" data-name="options">

The options parameter specifies the transfer option flags (for example,
I3C_OPT_RESPONSE).


</div>

<a name="return"></a>
### Return
<div id="Functions$I3C_write$description$return" data-type="text">

| Constants | Description |
| ----- | ----- |
| I3C_SUCCESS | Successful command submission |
| I3C_ERR_FIFO_FULL | Write or Command FIFO is full |
| I3C_ERR_NOT_CTRL | Not the active controller |
| I3C_ERR_PARAM | write_buffer is NULL with write_size > 0 |


</div>

##### Example
<div id="Functions$I3C_write$description$example$Example 1" data-type="text" data-name="Example">

</div>

<div id="Functions$I3C_write$description$example$Example 1" data-type="code" data-name="Example">


```
    uint8_t tx_buf[4] = {0xDE, 0xAD, 0xBE, 0xEF};
    I3C_write(&g_i3c, 0x12u, tx_buf, 4u,
              I3C_OPT_RESPONSE);
```

</div>


 -------------------------------- 
<a name="i3cread"></a>
## I3C_read
<a name="prototype"></a>
### Prototype 

<div id="Functions$I3C_read$prototype" data-type="code">

    i3c_status_t
    I3C_read
    (
        i3c_instance_t * this_i3c,
        uint8_t serial_addr,
        uint8_t * read_buffer,
        uint16_t read_size,
        uint32_t * read_len,
        uint32_t options
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$I3C_read$description" data-type="text">

The I3C_read() function initiates a private read transaction for an I3C controller. It
submits a read command descriptor to the Command FIFO and, after the transfer
completes, drains the Read FIFO into read_buffer. The function blocks until the
Read FIFO contains data or an error is detected.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_i3c
<div id="Functions$I3C_read$description$parameters$this_i3c" data-type="text" data-name="this_i3c">

The this_i3c parameter is a pointer to an i3c_instance_t structure that holds
all data related to the CoreI3C instance being initialized. A pointer to this
data structure is used in all subsequent calls to the CoreI3C driver functions
that operate on this CoreI3C instance.


</div>

#### serial_addr
<div id="Functions$I3C_read$description$parameters$serial_addr" data-type="text" data-name="serial_addr">

The serial_addr parameter specifies the 7-bit dynamic address of the target
device.


</div>

#### read_buffer
<div id="Functions$I3C_read$description$parameters$read_buffer" data-type="text" data-name="read_buffer">

The read_buffer parameter is a pointer to a buffer that receives the incoming data.
It must remain valid until the transfer completes.


</div>

#### read_size
<div id="Functions$I3C_read$description$parameters$read_size" data-type="text" data-name="read_size">

The read_size parameter specifies the maximum number of bytes to read into
read_buffer.


</div>

#### read_len
<div id="Functions$I3C_read$description$parameters$read_len" data-type="text" data-name="read_len">

The read_len parameter is a pointer to a uint32_t that receives the actual
number of bytes placed into read_buffer. It must not be NULL.


</div>

#### options
<div id="Functions$I3C_read$description$parameters$options" data-type="text" data-name="options">

The options parameter specifies the transfer option flags.


</div>

<a name="return"></a>
### Return
<div id="Functions$I3C_read$description$return" data-type="text">

| Constants | Description |
| ----- | ----- |
| I3C_SUCCESS | Successful completion |
| I3C_ERR_FIFO_FULL | Command FIFO is full |
| I3C_ERR_NOT_CTRL | Not the active controller |
| I3C_ERR_PARAM | read_buffer or read_len is NULL |


</div>

##### Example
<div id="Functions$I3C_read$description$example$Example 1" data-type="text" data-name="Example">

</div>

<div id="Functions$I3C_read$description$example$Example 1" data-type="code" data-name="Example">


```
    uint8_t rx_buf[8];
    uint32_t rx_len = 0u;
    I3C_read(&g_i3c, 0x12u, rx_buf, 8u, &rx_len,
             I3C_OPT_RESPONSE);
```

</div>


 -------------------------------- 
<a name="i3cwriteread"></a>
## I3C_write_read
<a name="prototype"></a>
### Prototype 

<div id="Functions$I3C_write_read$prototype" data-type="code">

    i3c_status_t
    I3C_write_read
    (
        i3c_instance_t * this_i3c,
        uint8_t serial_addr,
        const uint8_t * addr_offset,
        uint16_t offset_size,
        uint8_t * read_buffer,
        uint16_t read_size,
        uint32_t * read_len,
        uint32_t options
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$I3C_write_read$description" data-type="text">

The I3C_write_read() function initiates an I3C controller write-then-read
transaction. It first writes the addr_offset bytes to the target, then issues a
repeated-START and reads back read_size bytes into read_buffer. This is the
standard register read access pattern for I3C target devices.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_i3c
<div id="Functions$I3C_write_read$description$parameters$this_i3c" data-type="text" data-name="this_i3c">

The this_i3c parameter is a pointer to an i3c_instance_t structure that holds
all data related to the CoreI3C instance being initialized. A pointer to this
data structure is used in all subsequent calls to the CoreI3C driver functions
that operate on this CoreI3C instance.


</div>

#### serial_addr
<div id="Functions$I3C_write_read$description$parameters$serial_addr" data-type="text" data-name="serial_addr">

The serial_addr parameter specifies the 7-bit dynamic address of the target
device.


</div>

#### addr_offset
<div id="Functions$I3C_write_read$description$parameters$addr_offset" data-type="text" data-name="addr_offset">

The addr_offset parameter is a pointer to the register address/offset bytes to
write first.


</div>

#### offset_size
<div id="Functions$I3C_write_read$description$parameters$offset_size" data-type="text" data-name="offset_size">

The offset_size parameter specifies the number of bytes in addr_offset.


</div>

#### read_buffer
<div id="Functions$I3C_write_read$description$parameters$read_buffer" data-type="text" data-name="read_buffer">

The read_buffer parameter is a pointer to a buffer that receive the read data.


</div>

#### read_size
<div id="Functions$I3C_write_read$description$parameters$read_size" data-type="text" data-name="read_size">

The read_size parameter specifies the maximum number of bytes to read.


</div>

#### read_len
<div id="Functions$I3C_write_read$description$parameters$read_len" data-type="text" data-name="read_len">

The read_len parameter is a pointer to a uint32_t that receives the actual
number of bytes read.


</div>

#### options
<div id="Functions$I3C_write_read$description$parameters$options" data-type="text" data-name="options">

The options parameter specifies the transfer option flags.


</div>

<a name="return"></a>
### Return
<div id="Functions$I3C_write_read$description$return" data-type="text">

| Constants | Description |
| ----- | ----- |
| I3C_SUCCESS | Successful completion |
| I3C_ERR_PARAM | Any required pointer is NULL |
| I3C_ERR_FIFO_FULL | A FIFO is full |
| I3C_ERR_NOT_CTRL | Not the active controller |


</div>

##### Example
<div id="Functions$I3C_write_read$description$example$Example 1" data-type="text" data-name="Example">

</div>

<div id="Functions$I3C_write_read$description$example$Example 1" data-type="code" data-name="Example">


```
    uint8_t reg_addr[1] = {0x20u};
    uint8_t reg_data[2];
    uint32_t rx_len = 0u;
    I3C_write_read(&g_i3c, 0x12u, reg_addr, 1u,
                   reg_data, 2u, &rx_len, I3C_OPT_NONE);
```

</div>


 -------------------------------- 
<a name="i3creadresponse"></a>
## I3C_read_response
<a name="prototype"></a>
### Prototype 

<div id="Functions$I3C_read_response$prototype" data-type="code">

    uint32_t
    I3C_read_response
    (
        i3c_instance_t * this_i3c,
        uint32_t * response
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$I3C_read_response$description" data-type="text">

The I3C_read_response() function reads one response descriptor from the CoreI3C
Status FIFO. The response descriptor contains the transfer result, error type,
transaction ID and data length of the most recently completed command.

Call this function after receiving the I3C_CR_IRQ_RSPNS_RDY interrupt or after
polling I3C_get_irq_status() for the RSPNS_RDY bit.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_i3c
<div id="Functions$I3C_read_response$description$parameters$this_i3c" data-type="text" data-name="this_i3c">

The this_i3c parameter is a pointer to an i3c_instance_t structure that holds
all data related to the CoreI3C instance being initialized. A pointer to this
data structure is used in all subsequent calls to the CoreI3C driver functions
that operate on this CoreI3C instance.


</div>

#### response
<div id="Functions$I3C_read_response$description$parameters$response" data-type="text" data-name="response">

The response parameter is a pointer to a uint32_t to receive the raw 32-bit
response descriptor. It must not be NULL.


</div>

<a name="return"></a>
### Return
<div id="Functions$I3C_read_response$description$return" data-type="text">

Returns the number of response entries read (0 or 1). Returns 0 if the Status
FIFO is empty.


</div>

##### Example
<div id="Functions$I3C_read_response$description$example$Example 1" data-type="text" data-name="Example">

</div>

<div id="Functions$I3C_read_response$description$example$Example 1" data-type="code" data-name="Example">


```
    uint32_t resp = 0u;
    uint32_t n = I3C_read_response(&g_i3c, &resp);
    if (n > 0u)
    {
        uint8_t err = (resp >> 28u) & 0xFu;
        uint8_t tid = (resp >> 24u) & 0xFu;
    }
```

</div>


 -------------------------------- 
<a name="i3cgetstatus"></a>
## I3C_get_status
<a name="prototype"></a>
### Prototype 

<div id="Functions$I3C_get_status$prototype" data-type="code">

    uint32_t
    I3C_get_status
    (
        i3c_instance_t * this_i3c
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$I3C_get_status$description" data-type="text">

The I3C_get_status() function returns the current operational state of the
CoreI3C controller by reading the PRESENT_STATE and INTR_STAT registers. The
returned value includes the active controller flag, command sequence timeout
state and internal error flags.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_i3c
<div id="Functions$I3C_get_status$description$parameters$this_i3c" data-type="text" data-name="this_i3c">

The this_i3c parameter is a pointer to an i3c_instance_t structure that holds
all data related to the CoreI3C instance being initialized. A pointer to this
data structure is used in all subsequent calls to the CoreI3C driver functions
that operate on this CoreI3C instance.


</div>

<a name="return"></a>
### Return
<div id="Functions$I3C_get_status$description$return" data-type="text">

This function returns the 32-bit PRESENT_STATE register value. The caller can
inspect individual bits using the COREI3C_I3C_PRESENT_STATE_* constants from
corei3c_regs.h.


</div>

##### Example
<div id="Functions$I3C_get_status$description$example$Example 1" data-type="text" data-name="Example">

</div>

<div id="Functions$I3C_get_status$description$example$Example 1" data-type="code" data-name="Example">


```
    uint32_t state = I3C_get_status(&g_i3c);
    if (state & COREI3C_I3C_PRESENT_STATE_CURRENT_CTRL_MASK)
    {
        // This instance is the active controller
    }
```

</div>


 -------------------------------- 
<a name="i3cenableirq"></a>
## I3C_enable_irq
<a name="prototype"></a>
### Prototype 

<div id="Functions$I3C_enable_irq$prototype" data-type="code">

    void
    I3C_enable_irq
    (
        i3c_instance_t * this_i3c,
        uint32_t irq_mask
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$I3C_enable_irq$description" data-type="text">

The I3C_enable_irq() function enables CPU interrupt signaling (SIG_EN) for one
or more CoreI3C interrupt sources. When enabled, the INT_O output pin asserts to
the host CPU when the corresponding event occurs.

Status logging (STAT_EN) is always active after I3C_init_controller(). the
hardware always latches events into the status register regardless of whether
CPU signaling is enabled. This allows polling (using I3C_transfer_complete or
I3C_wait_complete) to work at any time without calling this function.

Call this function when interrupt-driven operation is desired.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_i3c
<div id="Functions$I3C_enable_irq$description$parameters$this_i3c" data-type="text" data-name="this_i3c">

The this_i3c parameter is a pointer to an i3c_instance_t structure that holds
all data related to the CoreI3C instance being initialized. A pointer to this
data structure is used in all subsequent calls to the CoreI3C driver functions
that operate on this CoreI3C instance.


</div>

#### irq_mask
<div id="Functions$I3C_enable_irq$description$parameters$irq_mask" data-type="text" data-name="irq_mask">

The irq_mask parameter specifies the bitmask of interrupt sources to enable. OR
of I3C_IRQ_* and I3C_CR_IRQ_* constants. Bits [15:0] correspond to general
INTR_SIG_EN. Bits [31:16] correspond to CMND_RSPNS_INTR_SIG_EN.


</div>

<a name="return"></a>
### Return
<div id="Functions$I3C_enable_irq$description$return" data-type="text">

This function does not return any value.


</div>

##### Example
<div id="Functions$I3C_enable_irq$description$example$Example 1" data-type="text" data-name="Example">

</div>

<div id="Functions$I3C_enable_irq$description$example$Example 1" data-type="code" data-name="Example">


```
    I3C_enable_irq(&g_i3c,
                   COREI3C_I3C_INTR_STAT_CMD_STALL_MASK    |
                   COREI3C_I3C_INTR_STAT_SEQ_CANCEL_MASK   |
                   COREI3C_I3C_INTR_STAT_INTERNAL_ERR_MASK |
                   I3C_CR_IRQ_CMND_RDY     |
                   I3C_CR_IRQ_RSPNS_RDY    |
                   I3C_CR_IRQ_IBI_RECV);
```

</div>


 -------------------------------- 
<a name="i3cdisableirq"></a>
## I3C_disable_irq
<a name="prototype"></a>
### Prototype 

<div id="Functions$I3C_disable_irq$prototype" data-type="code">

    void
    I3C_disable_irq
    (
        i3c_instance_t * this_i3c,
        uint32_t irq_mask
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$I3C_disable_irq$description" data-type="text">

The I3C_disable_irq() function disables CPU interrupt signaling (SIG_EN) for one
or more CoreI3C interrupt sources. The INT_O pin will no longer assert for the
specified events.

Status logging (STAT_EN) is NOT affected. the hardware continues to latch
events into the status register. Polling through I3C_transfer_complete() or
I3C_wait_complete() continues to work after calling this function.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_i3c
<div id="Functions$I3C_disable_irq$description$parameters$this_i3c" data-type="text" data-name="this_i3c">

The this_i3c parameter is a pointer to an i3c_instance_t structure that holds
all data related to the CoreI3C instance being initialized. A pointer to this
data structure is used in all subsequent calls to the CoreI3C driver functions
that operate on this CoreI3C instance.


</div>

#### irq_mask
<div id="Functions$I3C_disable_irq$description$parameters$irq_mask" data-type="text" data-name="irq_mask">

The irq_mask parameter specifies the bitmask of interrupt sources to disable. OR
of I3C_IRQ_* and I3C_CR_IRQ_* constants.


</div>

<a name="return"></a>
### Return
<div id="Functions$I3C_disable_irq$description$return" data-type="text">

This function does not return any value.


</div>

##### Example
<div id="Functions$I3C_disable_irq$description$example$Example 1" data-type="text" data-name="Example">

</div>

<div id="Functions$I3C_disable_irq$description$example$Example 1" data-type="code" data-name="Example">


```
    I3C_disable_irq(&g_i3c, I3C_CR_IRQ_WRT | I3C_CR_IRQ_RD);
```

</div>


 -------------------------------- 
<a name="i3cclearirq"></a>
## I3C_clear_irq
<a name="prototype"></a>
### Prototype 

<div id="Functions$I3C_clear_irq$prototype" data-type="code">

    void
    I3C_clear_irq
    (
        i3c_instance_t * this_i3c,
        uint32_t irq_mask
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$I3C_clear_irq$description" data-type="text">

The I3C_clear_irq() function clears one or more pending CoreI3C interrupt status
bits by writing 1 to the corresponding bit positions in the INTR_STAT and
CMND_RSPNS_INTR_STAT registers.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_i3c
<div id="Functions$I3C_clear_irq$description$parameters$this_i3c" data-type="text" data-name="this_i3c">

The this_i3c parameter is a pointer to an i3c_instance_t structure that holds
all data related to the CoreI3C instance being initialized. A pointer to this
data structure is used in all subsequent calls to the CoreI3C driver functions
that operate on this CoreI3C instance.


</div>

#### irq_mask
<div id="Functions$I3C_clear_irq$description$parameters$irq_mask" data-type="text" data-name="irq_mask">

The irq_mask parameter specifies the bitmask of interrupt sources to clear. OR
of I3C_IRQ_* and I3C_CR_IRQ_* constants.


</div>

<a name="return"></a>
### Return
<div id="Functions$I3C_clear_irq$description$return" data-type="text">

This function does not return any value.


</div>

##### Example
<div id="Functions$I3C_clear_irq$description$example$Example 1" data-type="text" data-name="Example">

</div>

<div id="Functions$I3C_clear_irq$description$example$Example 1" data-type="code" data-name="Example">


```
    void i3c_isr(void)
    {
        uint32_t pending = I3C_get_irq_status(&g_i3c);
        if (pending & I3C_CR_IRQ_CMND_RDY)
        {
            // handle command complete
            I3C_clear_irq(&g_i3c, I3C_CR_IRQ_CMND_RDY);
        }
    }
```

</div>


 -------------------------------- 
<a name="i3cgetirqstatus"></a>
## I3C_get_irq_status
<a name="prototype"></a>
### Prototype 

<div id="Functions$I3C_get_irq_status$prototype" data-type="code">

    uint32_t
    I3C_get_irq_status
    (
        i3c_instance_t * this_i3c
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$I3C_get_irq_status$description" data-type="text">

The I3C_get_irq_status() function returns the current combined interrupt status
of the CoreI3C controller. The return value encodes both the general interrupt
status (INTR_STAT register, bits [15:0]) and the command/response interrupt
status (CMND_RSPNS_INTR_STAT, bits [31:16]), matching the encoding used by
irq_mask in I3C_enable_irq()/I3C_clear_irq().

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_i3c
<div id="Functions$I3C_get_irq_status$description$parameters$this_i3c" data-type="text" data-name="this_i3c">

The this_i3c parameter is a pointer to an i3c_instance_t structure that holds
all data related to the CoreI3C instance being initialized. A pointer to this
data structure is used in all subsequent calls to the CoreI3C driver functions
that operate on this CoreI3C instance.


</div>

<a name="return"></a>
### Return
<div id="Functions$I3C_get_irq_status$description$return" data-type="text">

This function returns the combined 32-bit interrupt status. Compare with
I3C_IRQ_* and I3C_CR_IRQ_* constants to determine which sources are pending.


</div>

##### Example
<div id="Functions$I3C_get_irq_status$description$example$Example 1" data-type="text" data-name="Example">

</div>

<div id="Functions$I3C_get_irq_status$description$example$Example 1" data-type="code" data-name="Example">


```
    uint32_t status = I3C_get_irq_status(&g_i3c);
    if (status & I3C_CR_IRQ_RSPNS_RDY)
    {
        uint32_t resp;
        I3C_read_response(&g_i3c, &resp);
    }
```

</div>


 -------------------------------- 
<a name="i3csetdatentry"></a>
## I3C_set_dat_entry
<a name="prototype"></a>
### Prototype 

<div id="Functions$I3C_set_dat_entry$prototype" data-type="code">

    i3c_status_t
    I3C_set_dat_entry
    (
        i3c_instance_t * this_i3c,
        uint8_t dev_index,
        uint8_t dynamic_addr,
        uint8_t static_addr,
        uint8_t is_i2c
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$I3C_set_dat_entry$description" data-type="text">

The I3C_set_dat_entry() function writes a Device Address Table (DAT) entry for a
given target device. The DAT must be populated before any private transfer or
direct CCC can be issued to a target, because the hardware resolves target
addresses through the DAT using a 5-bit DEV_INDEX.

I3C_do_daa() automatically populates DAT entries for devices discovered during
ENTDAA. Use this function to manually configure DAT entries for:

  - I2C legacy devices on the bus
  - Targets assigned via SETDASA
  - Re-programming an entry after SETNEWDA

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_i3c
<div id="Functions$I3C_set_dat_entry$description$parameters$this_i3c" data-type="text" data-name="this_i3c">

The this_i3c parameter is a pointer to an i3c_instance_t structure that holds
all data related to the CoreI3C instance being initialized. A pointer to this
data structure is used in all subsequent calls to the CoreI3C driver functions
that operate on this CoreI3C instance.


</div>

#### dev_index
<div id="Functions$I3C_set_dat_entry$description$parameters$dev_index" data-type="text" data-name="dev_index">

The dev_index parameter specifies the DAT entry index (0..31). For targets
discovered through DAA, use the same index as the DCT entry index (0-based order of
discovery).


</div>

#### dynamic_addr
<div id="Functions$I3C_set_dat_entry$description$parameters$dynamic_addr" data-type="text" data-name="dynamic_addr">

The dynamic_addr parameter specifies the 7-bit dynamic address to store in the
DAT entry.


</div>

#### static_addr
<div id="Functions$I3C_set_dat_entry$description$parameters$static_addr" data-type="text" data-name="static_addr">

The static_addr parameter specifies the 7-bit static address (for I2C legacy
devices). Set it to 0 for I3C targets.


</div>

#### is_i2c
<div id="Functions$I3C_set_dat_entry$description$parameters$is_i2c" data-type="text" data-name="is_i2c">

The is_i2c parameter specifies whether this is a legacy I2C device. Set to 1 if
this is a legacy I2C device, or, 0 for I3C targets.


</div>

<a name="return"></a>
### Return
<div id="Functions$I3C_set_dat_entry$description$return" data-type="text">

| Constants | Description |
| ----- | ----- |
| I3C_SUCCESS | Success |
| I3C_ERR_PARAM | dev_index >= 32 or this_i3c is NULL |


</div>

##### Example
<div id="Functions$I3C_set_dat_entry$description$example$Example 1" data-type="text" data-name="Example">

</div>

<div id="Functions$I3C_set_dat_entry$description$example$Example 1" data-type="code" data-name="Example">


```
    // Register an I2C sensor at static address 0x48 in DAT slot 5
    I3C_set_dat_entry(&g_i3c, 5u, 0u, 0x48u, 1u);

    // Register an I3C target at dynamic address 0x12 in DAT slot 0
    I3C_set_dat_entry(&g_i3c, 0u, 0x12u, 0u, 0u);
```

</div>


 -------------------------------- 
<a name="i3ctransfercomplete"></a>
## I3C_transfer_complete
<a name="prototype"></a>
### Prototype 

<div id="Functions$I3C_transfer_complete$prototype" data-type="code">

    i3c_status_t
    I3C_transfer_complete
    (
        i3c_instance_t * this_i3c
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$I3C_transfer_complete$description" data-type="text">

The I3C_transfer_complete() function completes an in-flight read transfer. It
must be called either from the user ISR (interrupt-driven mode) or from a
polling loop (polled mode) after I3C_read() or I3C_read_ccc() returns
I3C_IN_PROGRESS.

This function:

 1. Checks CMND_RSPNS_INTR_STAT for RD_STAT (data ready) or TRANSFR_ERR.
 2. On success — drains the RD FIFO into the pending read buffer and sets the 
byte count through the pending_rd_len pointer.
 3. On error — clears the error, resumes the controller and returns the 
appropriate error code.
 4. Resets controller_status to I3C_SUCCESS (idle) when complete.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_i3c
<div id="Functions$I3C_transfer_complete$description$parameters$this_i3c" data-type="text" data-name="this_i3c">

The this_i3c parameter is a pointer to an i3c_instance_t structure that holds
all data related to the CoreI3C instance being initialized. A pointer to this
data structure is used in all subsequent calls to the CoreI3C driver functions
that operate on this CoreI3C instance.


</div>

<a name="return"></a>
### Return
<div id="Functions$I3C_transfer_complete$description$return" data-type="text">

| Constants | Description |
| ----- | ----- |
| I3C_SUCCESS | Transfer complete, data in caller buffer |
| I3C_IN_PROGRESS | Transfer not yet complete, call again |
| I3C_ERR_NACK | Transfer error occurred on bus |


</div>

##### Example
<div id="Functions$I3C_transfer_complete$description$example$Example 1" data-type="text" data-name="Example">

</div>

<div id="Functions$I3C_transfer_complete$description$example$Example 1" data-type="code" data-name="Example">


```
    // Polled mode
    i3c_status_t rc;
    do {
        rc = I3C_transfer_complete(&g_i3c);
    } while( I3C_IN_PROGRESS == rc );

    // Interrupt-driven mode — call from ISR
    void i3c_isr(void)
    {
        I3C_transfer_complete(&g_i3c);
    }
```

</div>


 -------------------------------- 
<a name="i3cgetdeviceinfo"></a>
## I3C_get_device_info
<a name="prototype"></a>
### Prototype 

<div id="Functions$I3C_get_device_info$prototype" data-type="code">

    i3c_status_t
    I3C_get_device_info
    (
        i3c_instance_t * this_i3c,
        i3c_dev_info_t * device,
        uint8_t index
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$I3C_get_device_info$description" data-type="text">

The I3C_get_device_info() function retrieves device characteristics for a given
DAT index.

For ENTDAA devices, the information is read directly from the DCT hardware
registers (live read — always reflects current hardware state). For
SETDASA/SETI2C devices, the information is read from the internal char_table
cache (populated from addr_list dev_info during I3C_do_daa()).

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_i3c
<div id="Functions$I3C_get_device_info$description$parameters$this_i3c" data-type="text" data-name="this_i3c">

The this_i3c parameter is a pointer to an i3c_instance_t structure that holds
all data related to the CoreI3C instance being initialized. A pointer to this
data structure is used in all subsequent calls to the CoreI3C driver functions
that operate on this CoreI3C instance.


</div>

#### device
<div id="Functions$I3C_get_device_info$description$parameters$device" data-type="text" data-name="device">

The device parameter is a pointer to an i3c_dev_info_t structure to populate.
It must not be NULL.


</div>

#### index
<div id="Functions$I3C_get_device_info$description$parameters$index" data-type="text" data-name="index">

The index parameter specifies the DAT entry index (0-based). It must be less than
this_i3c->dev_count.


</div>

<a name="return"></a>
### Return
<div id="Functions$I3C_get_device_info$description$return" data-type="text">

| Constants | Description |
| ----- | ----- |
| I3C_SUCCESS | Device info populated in *device |
| I3C_ERR_PARAM | NULL pointer or index out of range |


</div>

##### Example
<div id="Functions$I3C_get_device_info$description$example$Example 1" data-type="text" data-name="Example">

</div>

<div id="Functions$I3C_get_device_info$description$example$Example 1" data-type="code" data-name="Example">


```
    i3c_dev_info_t info;
    for (uint8_t i = 0; i < g_i3c.dev_count; i++)
    {
        I3C_get_device_info(&g_i3c, &info, i);
        // info.address, info.bcr, info.dcr, info.vendor_id, etc.
    }
```

</div>


 -------------------------------- 
<a name="i3cwaitcomplete"></a>
## I3C_wait_complete
<a name="prototype"></a>
### Prototype 

<div id="Functions$I3C_wait_complete$prototype" data-type="code">

    i3c_status_t
    I3C_wait_complete
    (
        i3c_instance_t * this_i3c,
        uint32_t timeout_count
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$I3C_wait_complete$description" data-type="text">

The I3C_wait_complete() function blocks until the current in-flight transfer
completes or the specified timeout expires. Internally it polls
I3C_transfer_complete() in a loop. This function waits while controller_status
is I3C_IN_PROGRESS.

This function works in both ISR-driven and polled modes:

  - ISR-driven: The ISR calls I3C_transfer_complete() which updates 
controller_status. This function detects the status change and returns.
  - Polled mode: Each iteration calls I3C_transfer_complete() which polls 
hardware and processes the completion.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_i3c
<div id="Functions$I3C_wait_complete$description$parameters$this_i3c" data-type="text" data-name="this_i3c">

The this_i3c parameter is a pointer to an i3c_instance_t structure that holds
all data related to the CoreI3C instance being initialized. A pointer to this
data structure is used in all subsequent calls to the CoreI3C driver functions
that operate on this CoreI3C instance.


</div>

#### timeout_count
<div id="Functions$I3C_wait_complete$description$parameters$timeout_count" data-type="text" data-name="timeout_count">

The timeout_count parameter specifies the maximum number of poll iterations
before returning I3C_ERR_TIMEOUT. Set it to 0 for the infinite wait (no timeout).


</div>

<a name="return"></a>
### Return
<div id="Functions$I3C_wait_complete$description$return" data-type="text">

The last_status value after transfer completion:

| Constants | Description |
| ----- | ----- |
| I3C_SUCCESS | Transfer completed successfully |
| I3C_ERR_NACK | Target did not acknowledge |
| I3C_ERR_ABORTED | Transfer was aborted |
| I3C_ERR_TIMEOUT | Timeout expired before completion |
| I3C_ERR_PARAM | this_i3c is NULL |


</div>

##### Example
<div id="Functions$I3C_wait_complete$description$example$Example 1" data-type="text" data-name="Example">

</div>

<div id="Functions$I3C_wait_complete$description$example$Example 1" data-type="code" data-name="Example">


```
    // Non-blocking write, then blocking wait
    I3C_write(&g_i3c, target_addr, buf, len, I3C_OPT_NONE);
    i3c_status_t rc = I3C_wait_complete(&g_i3c, 100000u);
    if (I3C_SUCCESS != rc)
    {
        // Handle error
    }

    // Non-blocking read, then blocking wait
    I3C_read(&g_i3c, target_addr, rx_buf, rx_len, &actual, I3C_OPT_NONE);
    rc = I3C_wait_complete(&g_i3c, 100000u);
```

</div>


 -------------------------------- 

