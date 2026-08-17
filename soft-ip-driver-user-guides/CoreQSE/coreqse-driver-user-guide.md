---
# CoreQSE Bare Metal Driver.
---

## Table of Contents 
  - [Introduction](#introduction)
     - [Features](#features)
  - [Driver Configuration](#driver-configuration)
  - [Theory of Operation](#theory-of-operation)
     - [Configuration](#configuration)
     - [Data Path Control](#data-path-control)
     - [Flow Control](#flow-control)
     - [Receive Filtering](#receive-filtering)
     - [Statistics](#statistics)
     - [Wake-on-LAN](#wake-on-lan)
     - [PCS Status](#pcs-status)
- [Types](#types)
  - [QSE_instance_t](#QSEinstancet)
  - [QSE_tx_config_t](#QSE_tx_configt)
  - [QSE_rx_config_t](#QSErxconfigt)
  - [QSE_pause_config_t](#QSEpauseconfigt)
  - [QSE_tx_stats_t](#QSE_tx_statst)
  - [QSE_rx_stats_t](#QSE_rx_statst)
- [Constants](#constants)
- [Functions](#functions)
  - [QSE_init](#QSEinit)
  - [QSE_tx_cfg_set](#QSE_tx_cfg_set)
  - [QSE_rx_cfg_set](#QSE_rx_cfg_set)
  - [QSE_set_mac_addr](#QSE_set_mac_addr)
  - [QSE_tx_enable](#QSE_tx_enable)
  - [QSE_tx_disable](#QSE_tx_disable)
  - [QSE_rx_enable](#QSE_rx_enable)
  - [QSE_rx_disable](#QSE_rx_disable)
  - [QSE_tx_frame](#QSE_tx_frame)
  - [QSE_get_tx_status](#QSE_get_tx_status)
  - [QSE_get_rx_status](#QSE_get_rx_status)
  - [QSE_pause_cfg_set](#QSE_pause_cfg_set)
  - [QSE_set_rx_filter_pkt_cfg](#QSE_set_rx_filter_pkt_cfg)
  - [QSE_set_rx_filter_hash](#QSE_set_rx_filter_hash)
  - [QSE_wol_enable](#QSE_wol_enable)
  - [QSE_wol_disable](#QSE_wol_disable)
  - [QSE_get_tx_stats](#QSE_get_tx_stats)
  - [QSE_get_rx_stats](#QSE_get_rx_stats)
  - [QSE_clear_stats](#QSE_clear_stats)
  - [QSE_pcs_get_sync_status](#QSE_pcs_get_sync_status)

<div id="TitlePage" data-type="text">

# Introduction
CoreQSE is an IP component that provides a Gigabit Ethernet Media Access 
Controller (MAC) with integrated Physical Coding Sublayer (PCS). CoreQSE 
implements IEEE® 802.3 compliant Ethernet frame transmission and reception at 
10/100/1000 Mbps speeds. The core provides AXI4-Stream interfaces for frame 
data transfer and an APB interface for configuration and status registers.

This driver provides a set of functions for controlling CoreQSE as part of the
bare metal system where no operating system is available. This driver can be
adapted to be used as a part of an operating system, but the implementation of
the adaptation layer between the driver and the operating system's driver model
is outside the scope of this driver.

## Features
The CoreQSE driver supports the following features:

  - Transmit and receive MAC path configuration
  - 48-bit MAC address programming
  - PAUSE flow control support with configurable watermarks
  - Packet-based receive filtering (FCS error, length, PCS error and so on.)
  - Hash-based destination MAC address filtering
  - Wake-on-LAN magic packet detection
  - Comprehensive transmit and receive statistics counters
  - PCS synchronization status monitoring
  - FCS insertion, removal and error injection
  - Configurable inter-packet gap and preamble

# Driver Configuration
Your application software must configure the CoreQSE driver by calling the
QSE_init() function for each CoreQSE instance in the hardware design. 
The QSE_init() function configures the CoreQSE hardware instance base address and
initializes the driver instance data structure. You must call the QSE_init() 
function before calling any other CoreQSE driver functions.

# Theory of Operation
The CoreQSE software driver controls multiple instances of CoreQSE IP. Each 
instance of CoreQSE in the hardware design is associated with a single instance 
of the QSE_instance_t structure in the software. You can initialize the contents 
of these data structures by calling the QSE_init() function. A pointer to the 
structure is passed to the subsequent driver functions to identify the CoreQSE 
hardware instance on which the operation should be performed.

Note: Do not attempt to directly manipulate the content of QSE_instance_t
structure. This structure must be modified by the driver function.

The CoreQSE driver operations are divided in following sub-sections:

  - Configuration
  - Data Path Control
  - Flow Control
  - Receive Filtering
  - Statistics
  - Wake-on-LAN
  - PCS Status

## Configuration
The CoreQSE driver must be initialized first before performing data transfers.
The QSE_init() function initializes the Ethernet MAC controller and driver. 
After initialization, configure the MAC address using the QSE_set_mac_addr() function.
Configure the transmit path using QSE_tx_cfg_set() function and the receive path
using QSE_rx_cfg_set() function. The transmit configuration includes FCS 
insertion mode, pad-enable for short frames, inter-packet gap and maximum packet 
length. The receive configuration includes FCS removal, pad removal and maximum 
receive packet length.

## Data Path Control
The QSE_tx_enable() function enables the MAC transmit data path, which allows 
Ethernet frames to flow from the AXI4-Stream TX interface to the GMII transmit 
interface. The QSE_tx_disable() function disables the transmission. Similarly, 
the QSE_rx_enable() and QSE_rx_disable() functions control the receive data path. 
The QSE_tx_frame() to transmit an Ethernet frame , use the GbEMR_tx_frame() 
function, which writes frame data to the transmit AXI4-Stream interface.

## Flow Control
The driver supports IEEE 802.3x PAUSE flow control. The QSE_pause_cfg_set() 
function configures the PAUSE mechanism, which includes receiving FIFO watermark 
levels, pause timer value and number of pause quanta before retransmitting a 
PAUSE frame. When the receive FIFO reaches the high watermark, the MAC 
automatically sends PAUSE frames to throttle the link partner's transmission.

## Receive Filtering
The QSE_set_rx_filter_pkt_cfg() function configures packet-based receive filtering.
Frames can be filtered based on FCS errors, length violations, PCS errors, unicast
DA mismatch, broadcast, multicast, VLAN tags, PAUSE frames, and unsupported control
opcodes. QSE_set_rx_filter_hash() function programs a 64-bit hash table for 
destination MAC address filtering. These filters take effect in store-and-forward
FIFO mode.

## Statistics
The CoreQSE driver provides functions to read transmit and receive statistics
counters. QSE_get_tx_stats() function reads all enabled transmit statistics 
counters into a QSE_tx_stats_t structure. QSE_get_rx_stats() function reads
receive statistics into a QSE_rx_stats_t structure. The counters use read-to-clear
behavior. QSE_clear_stats() function clears all the statistics counters by 
reading and discarding their values.

## Wake-on-LAN
The QSE_wol_enable() function enables Wake-on-LAN magic packet detection. The 
core scans incoming frames for the AMD Magic Packet pattern containing the 
configured MAC address repeated 16 times. When the system detects a magic packet, 
it asserts the WOL_DETECT signal for one clock cycle. The QSE_wol_disable() 
function disables this feature.

## PCS Status
The QSE_pcs_get_sync_status() function returns the synchronization state of the
Physical Coding Sublayer (PCS). It indicates whether the PCS has achieved a valid
lock state and the receive link is active.

</div>


# Types

 ---------------- 
<a name="QSEinstancet"></a>
## QSE_instance_t
<a name="prototype"></a>
### Prototype 

<div id="Types$QSE_instance_t$prototype" data-type="code">

``` 
    typedef struct QSE_instance {
        addr_t base_address;
    } QSE_instance_t;
``` 


</div>

<a name="description"></a>
### Description 

<div id="Types$QSE_instance_t$description" data-type="text">

This structure identifies various CoreQSE hardware instances within your system.
The QSE_init() function initializes this structure. You should provide a pointer 
to an initialized instance of the structure as the first parameter to the 
CoreQSE driver functions to identify which CoreQSE should perform the requested 
operation.


</div>


 --------------------------- 
<a name="QSE_tx_configt"></a>
## QSE_tx_config_t
<a name="prototype"></a>
### Prototype 

<div id="Types$QSE_tx_config_t$prototype" data-type="code">

``` 
    typedef struct QSE_tx_config {
        uint8_t fcs_insert_mode;
        uint8_t fcs_stomp_mode;
        uint8_t fcs_error_inject;
        uint8_t pad_enable;
        uint8_t ipg_count;
        uint16_t max_tx_pkt_len;
        uint8_t preamble_size;
        uint32_t preamble_content;
    } QSE_tx_config_t;
``` 


</div>

<a name="description"></a>
### Description 

<div id="Types$QSE_tx_config_t$description" data-type="text">

The QSE_tx_config_t structure is used to configure the transmit MAC path.
It contains configuration parameters for FCS handling, padding, inter-packet gap,
maximum packet length and preamble settings. This structure is passed to the
QSE_tx_cfg_set() function to set up the transmit path configuration.

| Field | Description |
| ----- | ----- |
| fcs_insert_mode | FCS insertion mode (0=no insert, 1=insert) |
| fcs_stomp_mode | FCS stomp mode for error injection |
| fcs_error_inject | Enable FCS error injection |
| pad_enable | Enable padding for short frames |
| ipg_count | Programmable inter-packet gap count |
| max_tx_pkt_len | Maximum transmit packet length |
| preamble_size | Custom preamble size |
| preamble_content | Custom preamble content |


</div>


 --------------------------- 
<a name="QSErxconfigt"></a>
## QSE_rx_config_t
<a name="prototype"></a>
### Prototype 

<div id="Types$QSE_rx_config_t$prototype" data-type="code">

``` 
    typedef struct QSE_rx_config {
        uint8_t fcs_remove;
        uint8_t pad_remove;
        uint16_t max_rx_pkt_len;
        uint8_t preamble_size;
    } QSE_rx_config_t;
``` 


</div>

<a name="description"></a>
### Description 

<div id="Types$QSE_rx_config_t$description" data-type="text">

The QSE_rx_config_t structure is used to configure the receive MAC path.
It contains configuration parameters for FCS removal, pad removal, maximum
receive packet length and preamble settings. This structure is passed to the
QSE_rx_cfg_set() function to set up the receive path configuration.

| Field | Description |
| ----- | ----- |
| fcs_remove | Enable FCS field removal |
| pad_remove | Enable pad byte removal for short frames |
| max_rx_pkt_len | Maximum receive packet length |
| preamble_size | Custom preamble size |


</div>


 --------------------------- 
<a name="QSEpauseconfigt"></a>
## QSE_pause_config_t
<a name="prototype"></a>
### Prototype 

<div id="Types$QSE_pause_config_t$prototype" data-type="code">

``` 
    typedef struct QSE_pause_config {
        uint16_t rx_fifo_high_watermark;
        uint16_t rx_fifo_low_watermark;
        uint16_t pause_timer;
        uint8_t pause_quanta_retransmit;
    } QSE_pause_config_t;
``` 


</div>

<a name="description"></a>
### Description 

<div id="Types$QSE_pause_config_t$description" data-type="text">

The QSE_pause_config_t structure configures the PAUSE flow control mechanism. It 
contains parameters for FIFO watermark levels and pause timing. This structure 
is passed to the QSE_pause_cfg_set() function.

| Field | Description |
| ----- | ----- |
| rx_fifo_high_watermark | Receive FIFO high-watermark level to trigger PAUSE |
| rx_fifo_low_watermark | Receive FIFO low-watermark level to stop PAUSE |
| pause_timer | 16-bit pause timer value (in units of 64 byte-times) |
| pause_quanta_retransmit | Number of pause quanta before retransmit PAUSE frame |


</div>


 --------------------------- 
<a name="QSE_tx_statst"></a>
## QSE_tx_stats_t
<a name="prototype"></a>
### Prototype 

<div id="Types$QSE_tx_stats_t$prototype" data-type="code">

``` 
    typedef struct QSE_tx_stats {
        uint32_t tx_frame_count;
        uint32_t tx_byte_count;
        uint32_t tx_underrun_count;
        uint32_t tx_pause_frame_count;
        /* Additional counters as enabled in hardware */
    } QSE_tx_stats_t;
``` 


</div>

<a name="description"></a>
### Description 

<div id="Types$QSE_tx_stats_t$description" data-type="text">

The QSE_tx_stats_t structure holds transmit statistics counter values. 
The QSE_get_tx_stats() function populates this structure. Counters not
enabled in the core configurator return zero. Each counter clears to zero in
hardware upon being read (read-to-clear behavior).


</div>


 --------------------------- 
<a name="QSE_rx_statst"></a>
## QSE_rx_stats_t
<a name="prototype"></a>
### Prototype 

<div id="Types$QSE_rx_stats_t$prototype" data-type="code">

``` 
    typedef struct QSE_rx_stats {
        uint32_t rx_frame_count;
        uint32_t rx_byte_count;
        uint32_t rx_overflow_count;
        uint32_t rx_fcs_error_count;
        uint32_t rx_pause_frame_count;
        /* Additional counters as enabled in hardware */
    } QSE_rx_stats_t;
``` 


</div>

<a name="description"></a>
### Description 

<div id="Types$QSE_rx_stats_t$description" data-type="text">

The QSE_rx_stats_t structure holds receive statistics counter values. 
The QSE_get_rx_stats() function populates this structure. The core configurator 
does not enable counters, so they return zero. Each counter clears to zero in
hardware upon being read (read-to-clear behavior).


</div>


 --------------------------- 

# Constants

 ---------------- 
<a name="constants"></a>

The following constants are defined for use with the CoreQSE driver:

## Filter Mask Constants
These constants are used with QSE_set_rx_filter_pkt_cfg() function to configure
receive frame filtering:

| Constants | Description |
| ----- | ----- |
| QSE_FILTER_FCS_ERROR | Filter frames with FCS errors |
| QSE_FILTER_LENGTH_LONG | Filter frames exceeding maximum length |
| QSE_FILTER_LENGTH_SHORT | Filter frames below minimum length |
| QSE_FILTER_LENGTH_CHECK | Filter frames failing length check |
| QSE_FILTER_PCS_ERROR | Filter frames with PCS errors |
| QSE_FILTER_UNICAST_DA_MISMATCH | Filter unicast frames not matching local MAC |
| QSE_FILTER_BROADCAST | Filter broadcast frames |
| QSE_FILTER_MULTICAST | Filter multicast frames |
| QSE_FILTER_VLAN | Filter VLAN-tagged frames |
| QSE_FILTER_PAUSE_OPCODE | Filter PAUSE frames |
| QSE_FILTER_CONTROL_FRAME | Filter control frames |
| QSE_FILTER_UNSUPPORTED_OPCODE | Filter unsupported control frames |

## Return Status Constants

| Constants | Description |
| ----- | ----- |
| QSE_OK | Operation completed successfully |
| QSE_ERR | General error condition |
| QSE_ERR_PARAM | Invalid parameter error |
| QSE_ERR_FIFO_FULL | Transmit FIFO is full |

 ---------------- 

# Functions

 ---------------- 
<a name="QSEinit"></a>
## QSE_init
<a name="prototype"></a>
### Prototype 

<div id="Functions$QSE_init$prototype" data-type="code">

    uint8_t
    QSE_init
    (
        QSE_instance_t * this_QSE,
        addr_t base_addr
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$QSE_init$description" data-type="text">

The QSE_init() function initializes the driver. It sets the base address for the
CoreQSE instance and initializes the QSE_instance_t data structure. You must 
call this function before calling any other CoreQSE driver functions.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_QSE
<div id="Functions$QSE_init$description$parameters$this_QSE" data-type="text" data-name="this_QSE">

The this_QSE parameter points to a QSE_instance_t structure that holds 
all data related to the CoreQSE instance being initialized. All subsequent calls 
to the CoreQSE driver functions use a pointer to this data structure to operate 
on this CoreGbE_MR.


</div>

#### base_addr
<div id="Functions$QSE_init$description$parameters$base_addr" data-type="text" data-name="base_addr">

The base_addr parameter is the base address in the processor's memory map for
the registers of the CoreQSE hardware instance being initialized.


</div>

<a name="return"></a>
### Return
<div id="Functions$QSE_init$description$return" data-type="text">

This function returns 0 on successful execution, otherwise, it will return
a non-zero error code.

| Constants | Description |
| ----- | ----- |
| 0 | Success |
| Non-zero | Error condition |


</div>

##### Example
<div id="Functions$QSE_init$description$example$Example1" data-type="text" data-name="Example">
Basic initialization of the CoreQSE driver.


</div>

<div id="Functions$QSE_init$description$example$Example1" data-type="code" data-name="Example">


```
    QSE_instance_t g_QSE;
    
    int main(void)
    {
        uint8_t status;
        
        status = QSE_init(&g_QSE, CoreQSE_BASE_ADDR);
        if (status != 0)
        {
            /* Handle initialization error */
            return -1;
        }

        return 0;
    }
```

</div>


 -------------------------------- 
<a name="QSE_tx_cfg_set"></a>
## QSE_tx_cfg_set
<a name="prototype"></a>
### Prototype 

<div id="Functions$QSE_tx_cfg_set$prototype" data-type="code">

    uint8_t
    QSE_tx_cfg_set
    (
        QSE_instance_t * this_QSE,
        const QSE_tx_config_t * cfg
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$QSE_tx_cfg_set$description" data-type="text">

The QSE_tx_cfg_set() function configures the transmit MAC path. It sets up
FCS insertion mode, FCS stomp and error injection modes, pad-enable for short 
frames, programmable inter-packet gap count, maximum transmit packet length 
and custom preamble content and size.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_QSE
<div id="Functions$QSE_tx_cfg_set$description$parameters$this_QSE" data-type="text" data-name="this_QSE">

The this_QSE parameter points to a QSE_instance_t structure that holds 
all data related to the CoreQSE instance being configured.


</div>

#### cfg
<div id="Functions$QSE_tx_cfg_set$description$parameters$cfg" data-type="text" data-name="cfg">

The cfg parameter points to a QSE_tx_config_t structure containing the
transmit configuration settings.


</div>

<a name="return"></a>
### Return
<div id="Functions$QSE_tx_cfg_set$description$return" data-type="text">

This function returns 0 on successful execution, otherwise, it will return
a non-zero error code if any parameter is out of range.


</div>

##### Example 
<div id="Functions$QSE_tx_cfg_set$description$example$Example1" data-type="text" data-name="Example">

</div>

<div id="Functions$QSE_tx_cfg_set$description$example$Example1" data-type="code" data-name="Example">


```
    QSE_instance_t g_QSE;
    QSE_tx_config_t tx_cfg;
    
    int main(void)
    {
        QSE_init(&g_QSE, CoreQSE_BASE_ADDR);
        
        /* Configure TX path */
        tx_cfg.fcs_insert_mode = 1;    /* Enable FCS insertion */
        tx_cfg.pad_enable = 1;         /* Enable padding */
        tx_cfg.ipg_count = 12;         /* 12 byte IPG */
        tx_cfg.max_tx_pkt_len = 1518;  /* Standard MTU + headers */
        
        QSE_tx_cfg_set(&g_QSE, &tx_cfg);
        
        return 0;
    }
```

</div>


 -------------------------------- 
<a name="QSE_rx_cfg_set"></a>
## QSE_rx_cfg_set
<a name="prototype"></a>
### Prototype 

<div id="Functions$QSE_rx_cfg_set$prototype" data-type="code">

    uint8_t
    QSE_rx_cfg_set
    (
        QSE_instance_t * this_QSE,
        const QSE_rx_config_t * cfg
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$QSE_rx_cfg_set$description" data-type="text">

The QSE_rx_cfg_set() function configures the receive MAC path. It sets up
FCS field removal, pad byte removal for short frames, maximum receive packet
length and custom preamble size.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_QSE
<div id="Functions$QSE_rx_cfg_set$description$parameters$this_QSE" data-type="text" data-name="this_QSE">

The this_QSE parameter points to a QSE_instance_t structure that holds 
all data related to the CoreQSE instance being configured.


</div>

#### cfg
<div id="Functions$QSE_rx_cfg_set$description$parameters$cfg" data-type="text" data-name="cfg">

The cfg parameter points to a QSE_rx_config_t structure containing the
receive configuration settings.


</div>

<a name="return"></a>
### Return
<div id="Functions$QSE_rx_cfg_set$description$return" data-type="text">

This function returns 0 on successful execution, otherwise, it will return
a non-zero error code if any parameter is out of range.


</div>

##### Example 
<div id="Functions$QSE_rx_cfg_set$description$example$Example1" data-type="text" data-name="Example">

</div>

<div id="Functions$QSE_rx_cfg_set$description$example$Example1" data-type="code" data-name="Example">


```
    QSE_instance_t g_QSE;
    QSE_rx_config_t rx_cfg;
    
    int main(void)
    {
        QSE_init(&g_QSE, CoreQSE_BASE_ADDR);
        
        /* Configure RX path */
        rx_cfg.fcs_remove = 1;         /* Remove FCS from received frames */
        rx_cfg.pad_remove = 0;         /* Keep pad bytes */
        rx_cfg.max_rx_pkt_len = 1518;  /* Standard MTU + headers */
        
        QSE_rx_cfg_set(&g_QSE, &rx_cfg);
        
        return 0;
    }
```

</div>


 -------------------------------- 
<a name="QSE_set_mac_addr"></a>
## QSE_set_mac_addr
<a name="prototype"></a>
### Prototype 

<div id="Functions$QSE_set_mac_addr$prototype" data-type="code">

    uint8_t
    QSE_set_mac_addr
    (
        QSE_instance_t * this_QSE,
        const uint8_t * mac_addr
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$QSE_set_mac_addr$description" data-type="text">

The QSE_set_mac_addr() function programs the 48-bit MAC address into the 
hardware. The system uses the MAC address as the source address in transmitted 
frames, as a destination address reference for PAUSE frame detection, and for 
Wake-on-LAN magic packet matching.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_QSE
<div id="Functions$QSE_set_mac_addr$description$parameters$this_QSE" data-type="text" data-name="this_QSE">

The this_QSE parameter points to a QSE_instance_t structure that holds 
all data related to the CoreQSE instance.


</div>

#### mac_addr
<div id="Functions$QSE_set_mac_addr$description$parameters$mac_addr" data-type="text" data-name="mac_addr">

The mac_addr parameter points to a 6-byte array containing the MAC address
in network byte order (most significant byte first).


</div>

<a name="return"></a>
### Return
<div id="Functions$QSE_set_mac_addr$description$return" data-type="text">

This function returns 0 on success.


</div>

##### Example 
<div id="Functions$QSE_set_mac_addr$description$example$Example1" data-type="text" data-name="Example">

</div>

<div id="Functions$QSE_set_mac_addr$description$example$Example1" data-type="code" data-name="Example">


```
    QSE_instance_t g_QSE;
    
    int main(void)
    {
        uint8_t mac_addr[6] = {0x00, 0x11, 0x22, 0x33, 0x44, 0x55};
        
        QSE_init(&g_QSE, CoreQSE_BASE_ADDR);
        QSE_set_mac_addr(&g_QSE, mac_addr);
        
        return 0;
    }
```

</div>


 -------------------------------- 
<a name="QSE_tx_enable"></a>
## QSE_tx_enable
<a name="prototype"></a>
### Prototype 

<div id="Functions$QSE_tx_enable$prototype" data-type="code">

    void
    QSE_tx_enable
    (
        QSE_instance_t * this_QSE
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$QSE_tx_enable$description" data-type="text">

The QSE_tx_enable() function enables the MAC transmit data path. It allows
Ethernet frames to flow from the AXI4-Stream TX interface to the GMII transmit
interface.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_QSE
<div id="Functions$QSE_tx_enable$description$parameters$this_QSE" data-type="text" data-name="this_QSE">

The this_QSE parameter points to a QSE_instance_t structure that holds 
all data related to the CoreQSE instance.


</div>

<a name="return"></a>
### Return
<div id="Functions$QSE_tx_enable$description$return" data-type="text">

This function does not return a value.


</div>

##### Example 
<div id="Functions$QSE_tx_enable$description$example$Example1" data-type="text" data-name="Example">

</div>

<div id="Functions$QSE_tx_enable$description$example$Example1" data-type="code" data-name="Example">


```
    int main(void)
    {
        QSE_init(&g_QSE, CoreQSE_BASE_ADDR);
        QSE_set_mac_addr(&g_QSE, mac_addr);
        QSE_tx_enable(&g_QSE);
        
        ....
        
        return 0;
    }
```

</div>


 -------------------------------- 
<a name="QSE_tx_disable"></a>
## QSE_tx_disable
<a name="prototype"></a>
### Prototype 

<div id="Functions$QSE_tx_disable$prototype" data-type="code">

    void
    QSE_tx_disable
    (
        QSE_instance_t * this_QSE
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$QSE_tx_disable$description" data-type="text">

The QSE_tx_disable() function disables the MAC transmit data path. It prevents
Ethernet frames from being transmitted. The transition takes effect on a packet
boundary.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_QSE
<div id="Functions$QSE_tx_disable$description$parameters$this_QSE" data-type="text" data-name="this_QSE">

The this_QSE parameter points to a QSE_instance_t structure that holds 
all data related to the CoreQSE instance.


</div>

<a name="return"></a>
### Return
<div id="Functions$QSE_tx_disable$description$return" data-type="text">

This function does not return a value.


</div>

##### Example 
<div id="Functions$QSE_tx_disable$description$example$Example1" data-type="text" data-name="Example">

</div>

<div id="Functions$QSE_tx_disable$description$example$Example1" data-type="code" data-name="Example">


```
    int main(void)
    {
        QSE_init(&g_QSE, CoreQSE_BASE_ADDR);
        QSE_tx_enable(&g_QSE);
        
        ....
        
        QSE_tx_disable(&g_QSE);
        
        return 0;
    }
```

</div>


 -------------------------------- 
<a name="QSE_rx_enable"></a>
## QSE_rx_enable
<a name="prototype"></a>
### Prototype 

<div id="Functions$QSE_rx_enable$prototype" data-type="code">

    void
    QSE_rx_enable
    (
        QSE_instance_t * this_QSE
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$QSE_rx_enable$description" data-type="text">

The QSE_rx_enable() function enables the MAC receive data path. It allows
incoming Ethernet frames from the GMII interface to be delivered to the 
AXI4-Stream RX interface.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_QSE
<div id="Functions$QSE_rx_enable$description$parameters$this_QSE" data-type="text" data-name="this_QSE">

The this_QSE parameter points to a QSE_instance_t structure that holds 
all data related to the CoreQSE instance.


</div>

<a name="return"></a>
### Return
<div id="Functions$QSE_rx_enable$description$return" data-type="text">

This function does not return a value.


</div>

##### Example 
<div id="Functions$QSE_rx_enable$description$example$Example1" data-type="text" data-name="Example">

</div>

<div id="Functions$QSE_rx_enable$description$example$Example1" data-type="code" data-name="Example">


```
    int main(void)
    {
        QSE_init(&g_QSE, CoreQSE_BASE_ADDR);
        QSE_set_mac_addr(&g_QSE, mac_addr);
        QSE_rx_enable(&g_QSE);
        
        ....
        
        return 0;
    }
```

</div>


 -------------------------------- 
<a name="QSE_rx_disable"></a>
## QSE_rx_disable
<a name="prototype"></a>
### Prototype 

<div id="Functions$QSE_rx_disable$prototype" data-type="code">

    void
    QSE_rx_disable
    (
        QSE_instance_t * this_QSE
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$QSE_rx_disable$description" data-type="text">

The QSE_rx_disable() function disables the MAC receive data path. It prevents
incoming frames from being forwarded to the application.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_QSE
<div id="Functions$QSE_rx_disable$description$parameters$this_QSE" data-type="text" data-name="this_QSE">

The this_QSE parameter points to a QSE_instance_t structure that holds 
all data related to the CoreQSE instance.


</div>

<a name="return"></a>
### Return
<div id="Functions$QSE_rx_disable$description$return" data-type="text">

This function does not return a value.


</div>

##### Example 
<div id="Functions$QSE_rx_disable$description$example$Example1" data-type="text" data-name="Example">

</div>

<div id="Functions$QSE_rx_disable$description$example$Example1" data-type="code" data-name="Example">


```
    int main(void)
    {
        QSE_init(&g_QSE, CoreQSE_BASE_ADDR);
        QSE_rx_enable(&g_QSE);
        
        ....
        
        QSE_rx_disable(&g_QSE);
        
        return 0;
    }
```

</div>


 -------------------------------- 
<a name="QSE_tx_frame"></a>
## QSE_tx_frame
<a name="prototype"></a>
### Prototype 

<div id="Functions$QSE_tx_frame$prototype" data-type="code">

    uint8_t
    QSE_tx_frame
    (
        QSE_instance_t * this_QSE,
        const uint8_t * frame_data,
        uint32_t frame_len
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$QSE_tx_frame$description" data-type="text">

The QSE_tx_frame() function initiates transmission of one Ethernet frame.
It writes the frame payload to the transmit AXI4-Stream interface, asserting
TVALID with provided data and byte-enable qualifiers for each beat, and 
asserts TLAST at the final beat.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_QSE
<div id="Functions$QSE_tx_frame$description$parameters$this_QSE" data-type="text" data-name="this_QSE">

The this_QSE parameter points to a QSE_instance_t structure that holds 
all data related to the CoreQSE instance.


</div>

#### frame_data
<div id="Functions$QSE_tx_frame$description$parameters$frame_data" data-type="text" data-name="frame_data">

The frame_data parameter points to the frame payload data to be transmitted.


</div>

#### frame_len
<div id="Functions$QSE_tx_frame$description$parameters$frame_len" data-type="text" data-name="frame_len">

The frame_len parameter specifies the length of the frame in bytes.


</div>

<a name="return"></a>
### Return
<div id="Functions$QSE_tx_frame$description$return" data-type="text">

This function returns 0 on successful execution, otherwise, it will return
a non-zero error code if the transmit FIFO is full or frame_len is zero.


</div>

##### Example 
<div id="Functions$QSE_tx_frame$description$example$Example1" data-type="text" data-name="Example">

</div>

<div id="Functions$QSE_tx_frame$description$example$Example1" data-type="code" data-name="Example">


```
    QSE_instance_t g_QSE;
    
    int main(void)
    {
        uint8_t frame[64];
        uint8_t status;
        
        QSE_init(&g_QSE, CoreQSE_BASE_ADDR);
        QSE_tx_enable(&g_QSE);
        
        /* Prepare frame data */
        /* ... populate frame data ... */
        
        status = QSE_tx_frame(&g_QSE, frame, sizeof(frame));
        if (status != 0)
        {
            /* Handle transmission error */
        }
        
        return 0;
    }
```

</div>


 -------------------------------- 
<a name="QSE_get_tx_status"></a>
## QSE_get_tx_status
<a name="prototype"></a>
### Prototype 

<div id="Functions$QSE_get_tx_status$prototype" data-type="code">

    uint8_t
    QSE_get_tx_status
    (
        QSE_instance_t * this_QSE
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$QSE_get_tx_status$description" data-type="text">

The QSE_get_tx_status() function returns the current transmit error status.
It provides a bitmask indicating transmit error conditions such as transmit
FIFO underrun flag and other active transmit error indicators.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_QSE
<div id="Functions$QSE_get_tx_status$description$parameters$this_QSE" data-type="text" data-name="this_QSE">

The this_QSE parameter points to a QSE_instance_t structure that holds 
all data related to the CoreQSE instance.


</div>

<a name="return"></a>
### Return
<div id="Functions$QSE_get_tx_status$description$return" data-type="text">

This function returns a bitmask indicating transmit error status. A 0 value 
indicates no transmit error is pending.


</div>

##### Example 
<div id="Functions$QSE_get_tx_status$description$example$Example1" data-type="text" data-name="Example">

</div>

<div id="Functions$QSE_get_tx_status$description$example$Example1" data-type="code" data-name="Example">


```
    int main(void)
    {
        uint8_t tx_status;
        
        ....
        
        tx_status = QSE_get_tx_status(&g_QSE);
        if (tx_status != 0)
        {
            /* Handle transmit error */
        }
        
        return 0;
    }
```

</div>


 -------------------------------- 
<a name="QSE_get_rx_status"></a>
## QSE_get_rx_status
<a name="prototype"></a>
### Prototype 

<div id="Functions$QSE_get_rx_status$prototype" data-type="code">

    uint8_t
    QSE_get_rx_status
    (
        QSE_instance_t * this_QSE
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$QSE_get_rx_status$description" data-type="text">

The QSE_get_rx_status() function returns the current receive error status.
It provides a bitmask indicating receive error conditions such as receive
FIFO overflow flag and other active receive error indicators.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_QSE
<div id="Functions$QSE_get_rx_status$description$parameters$this_QSE" data-type="text" data-name="this_QSE">

The this_QSE parameter points to a QSE_instance_t structure that holds 
all data related to the CoreQSE instance.


</div>

<a name="return"></a>
### Return
<div id="Functions$QSE_get_rx_status$description$return" data-type="text">

This function returns a bitmask indicating receive error status. A 0 value 
indicates no receive error is pending.


</div>

##### Example 
<div id="Functions$QSE_get_rx_status$description$example$Example1" data-type="text" data-name="Example">

</div>

<div id="Functions$QSE_get_rx_status$description$example$Example1" data-type="code" data-name="Example">


```
    int main(void)
    {
        uint8_t rx_status;
        
        ....
        
        rx_status = QSE_get_rx_status(&g_QSE);
        if (rx_status != 0)
        {
            /* Handle receive error */
        }
        
        return 0;
    }
```

</div>


 -------------------------------- 
<a name="QSE_pause_cfg_set"></a>
## QSE_pause_cfg_set
<a name="prototype"></a>
### Prototype 

<div id="Functions$QSE_pause_cfg_set$prototype" data-type="code">

    uint8_t
    QSE_pause_cfg_set
    (
        QSE_instance_t * this_QSE,
        const QSE_pause_config_t * cfg
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$QSE_pause_cfg_set$description" data-type="text">

The QSE_pause_cfg_set() function configures the PAUSE flow control mechanism.
It sets the receive FIFO high-watermark level, receive FIFO low-watermark level,
16-bit pause timer value (in units of 64 byte-times) and number of pause quanta
before retransmitting a PAUSE frame.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_QSE
<div id="Functions$QSE_pause_cfg_set$description$parameters$this_QSE" data-type="text" data-name="this_QSE">

The this_QSE parameter points to a QSE_instance_t structure that holds 
all data related to the CoreQSE instance.


</div>

#### cfg
<div id="Functions$QSE_pause_cfg_set$description$parameters$cfg" data-type="text" data-name="cfg">

The cfg parameter points to a QSE_pause_config_t structure containing
the PAUSE flow control configuration settings.


</div>

<a name="return"></a>
### Return
<div id="Functions$QSE_pause_cfg_set$description$return" data-type="text">

This function returns 0 on success.


</div>

##### Example 
<div id="Functions$QSE_pause_cfg_set$description$example$Example1" data-type="text" data-name="Example">

</div>

<div id="Functions$QSE_pause_cfg_set$description$example$Example1" data-type="code" data-name="Example">


```
    QSE_instance_t g_QSE;
    QSE_pause_config_t pause_cfg;
    
    int main(void)
    {
        QSE_init(&g_QSE, CoreQSE_BASE_ADDR);
        
        /* Configure PAUSE flow control */
        pause_cfg.rx_fifo_high_watermark = 0x800;
        pause_cfg.rx_fifo_low_watermark = 0x400;
        pause_cfg.pause_timer = 0xFFFF;
        pause_cfg.pause_quanta_retransmit = 2;
        
        QSE_pause_cfg_set(&g_QSE, &pause_cfg);
        
        return 0;
    }
```

</div>


 -------------------------------- 
<a name="QSE_set_rx_filter_pkt_cfg"></a>
## QSE_set_rx_filter_pkt_cfg
<a name="prototype"></a>
### Prototype 

<div id="Functions$QSE_set_rx_filter_pkt_cfg$prototype" data-type="code">

    uint8_t
    QSE_set_rx_filter_pkt_cfg
    (
        QSE_instance_t * this_QSE,
        uint32_t filter_mask
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$QSE_set_rx_filter_pkt_cfg$description" data-type="text">

The QSE_set_rx_filter_pkt_cfg() function configures packet-based receive frame
filtering. Each bit in the filter_mask parameter enables dropping of frames
matching the corresponding condition.

NOTE: This filter takes effect only in store-and-forward FIFO mode.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_QSE
<div id="Functions$QSE_set_rx_filter_pkt_cfg$description$parameters$this_QSE" data-type="text" data-name="this_QSE">

The this_QSE parameter points to a QSE_instance_t structure that holds 
all data related to the CoreQSE instance.


</div>

#### filter_mask
<div id="Functions$QSE_set_rx_filter_pkt_cfg$description$parameters$filter_mask" data-type="text" data-name="filter_mask">

The filter_mask parameter is a bitmask that enables frame filtering conditions.
The following filter conditions are available:

| Condition | Description |
| ----- | ----- |
| QSE_FILTER_FCS_ERROR | Frames with FCS errors |
| QSE_FILTER_LENGTH_LONG | Frames exceeding maximum length |
| QSE_FILTER_LENGTH_SHORT | Frames below minimum length |
| QSE_FILTER_LENGTH_CHECK | Frames failing length check |
| QSE_FILTER_PCS_ERROR | Frames with PCS errors |
| QSE_FILTER_UNICAST_DA_MISMATCH | Unicast frames not matching local MAC |
| QSE_FILTER_BROADCAST | Broadcast frames |
| QSE_FILTER_MULTICAST | Multicast frames |
| QSE_FILTER_VLAN | VLAN-tagged frames |
| QSE_FILTER_PAUSE_OPCODE | PAUSE frames |
| QSE_FILTER_CONTROL_FRAME | Control frames |
| QSE_FILTER_UNSUPPORTED_OPCODE | Unsupported control frames |


</div>

<a name="return"></a>
### Return
<div id="Functions$QSE_set_rx_filter_pkt_cfg$description$return" data-type="text">

This function returns 0 on success.


</div>

##### Example 
<div id="Functions$QSE_set_rx_filter_pkt_cfg$description$example$Example1" data-type="text" data-name="Example">

</div>

<div id="Functions$QSE_set_rx_filter_pkt_cfg$description$example$Example1" data-type="code" data-name="Example">


```
    QSE_instance_t g_QSE;
    
    void setup_rx_filtering(void)
    {
        uint32_t filter_mask = 0;
        
        /* Enable filtering of frames with FCS errors and oversized frames */
        filter_mask |= QSE_FILTER_FCS_ERROR;
        filter_mask |= QSE_FILTER_LENGTH_LONG;
        
        QSE_set_rx_filter_pkt_cfg(&g_QSE, filter_mask);
    }
```

</div>


 -------------------------------- 
<a name="QSE_set_rx_filter_hash"></a>
## QSE_set_rx_filter_hash
<a name="prototype"></a>
### Prototype 

<div id="Functions$QSE_set_rx_filter_hash$prototype" data-type="code">

    uint8_t
    QSE_set_rx_filter_hash
    (
        QSE_instance_t * this_QSE,
        uint32_t hash_table_low,
        uint32_t hash_table_high
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$QSE_set_rx_filter_hash$description" data-type="text">

The QSE_set_rx_filter_hash() function programs the 64-bit hash table for 
destination MAC address filtering. Frames with destination MAC addresses mapped 
to a cleared hash entry are filtered out.

To use this feature:
1. Compute the 6-bit hash index by XOR-folding every sixth bit of the 48-bit MAC address
2. Set the corresponding bit in the 64-bit hash table for the desired MAC addresses

NOTE: This filter takes effect only in store-and-forward FIFO mode.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_QSE
<div id="Functions$QSE_set_rx_filter_hash$description$parameters$this_QSE" data-type="text" data-name="this_QSE">

The this_QSE parameter points to a QSE_instance_t structure that holds 
all data related to the CoreQSE instance.


</div>

#### hash_table_low
<div id="Functions$QSE_set_rx_filter_hash$description$parameters$hash_table_low" data-type="text" data-name="hash_table_low">

The hash_table_low parameter contains the lower 32 bits of the 64-bit hash table.


</div>

#### hash_table_high
<div id="Functions$QSE_set_rx_filter_hash$description$parameters$hash_table_high" data-type="text" data-name="hash_table_high">

The hash_table_high parameter contains the upper 32 bits of the 64-bit hash table.


</div>

<a name="return"></a>
### Return
<div id="Functions$QSE_set_rx_filter_hash$description$return" data-type="text">

This function returns 0 on success.


</div>

##### Example 
<div id="Functions$QSE_set_rx_filter_hash$description$example$Example1" data-type="text" data-name="Example">
Computing hash index and configuring hash table.


</div>

<div id="Functions$QSE_set_rx_filter_hash$description$example$Example1" data-type="code" data-name="Example">


```
    QSE_instance_t g_QSE;
    
    /* Compute 6-bit hash index from MAC address */
    uint8_t compute_hash_index(const uint8_t *mac_addr)
    {
        uint8_t hash = 0;
        for (int i = 0; i < 6; i++)
        {
            hash ^= mac_addr[i];
        }
        return hash & 0x3F;  /* 6-bit result */
    }
    
    void setup_hash_filter(void)
    {
        uint32_t hash_low = 0x00000001;   /* Accept MACs hashing to bit 0 */
        uint32_t hash_high = 0x00000000;
        
        QSE_set_rx_filter_hash(&g_QSE, hash_low, hash_high);
    }
```

</div>


 -------------------------------- 
<a name="QSE_wol_enable"></a>
## QSE_wol_enable
<a name="prototype"></a>
### Prototype 

<div id="Functions$QSE_wol_enable$prototype" data-type="code">

    void
    QSE_wol_enable
    (
        QSE_instance_t * this_QSE
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$QSE_wol_enable$description" data-type="text">

The QSE_wol_enable() function enables Wake-on-LAN magic packet detection.
The core scans incoming frames for the AMD Magic Packet pattern containing the
configured MAC address repeated 16 times. When a magic packet is detected, the
WOL_DETECT signal is asserted for one clock cycle.

NOTE: The MAC address must be set using QSE_set_mac_addr() before enabling WoL.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_QSE
<div id="Functions$QSE_wol_enable$description$parameters$this_QSE" data-type="text" data-name="this_QSE">

The this_QSE parameter points to a QSE_instance_t structure that holds 
all data related to the CoreQSE instance.


</div>

<a name="return"></a>
### Return
<div id="Functions$QSE_wol_enable$description$return" data-type="text">

This function does not return a value.


</div>

##### Example 
<div id="Functions$QSE_wol_enable$description$example$Example1" data-type="text" data-name="Example">

</div>

<div id="Functions$QSE_wol_enable$description$example$Example1" data-type="code" data-name="Example">


```
    QSE_instance_t g_QSE;
    
    void setup_wake_on_lan(void)
    {
        uint8_t mac_addr[6] = {0x00, 0x11, 0x22, 0x33, 0x44, 0x55};
        
        /* Set MAC address for WoL detection */
        QSE_set_mac_addr(&g_QSE, mac_addr);
        
        /* Enable Wake-on-LAN */
        QSE_wol_enable(&g_QSE);
        
        /* System can now enter low-power state */
        /* WOL_DETECT signal will wake the system on magic packet */
    }
```

</div>


 -------------------------------- 
<a name="QSE_wol_disable"></a>
## QSE_wol_disable
<a name="prototype"></a>
### Prototype 

<div id="Functions$QSE_wol_disable$prototype" data-type="code">

    void
    QSE_wol_disable
    (
        QSE_instance_t * this_QSE
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$QSE_wol_disable$description" data-type="text">

The QSE_wol_disable() function disables Wake-on-LAN magic packet detection.
It stops scanning for magic packet pattern and the WOL_DETECT signal remains
de-asserted.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_QSE
<div id="Functions$QSE_wol_disable$description$parameters$this_QSE" data-type="text" data-name="this_QSE">

The this_QSE parameter points to a QSE_instance_t structure that holds 
all data related to the CoreQSE instance.


</div>

<a name="return"></a>
### Return
<div id="Functions$QSE_wol_disable$description$return" data-type="text">

This function does not return a value.


</div>

##### Example 
<div id="Functions$QSE_wol_disable$description$example$Example1" data-type="text" data-name="Example">

</div>

<div id="Functions$QSE_wol_disable$description$example$Example1" data-type="code" data-name="Example">


```
    int main(void)
    {
        ....
        
        /* Disable Wake-on-LAN when not needed */
        QSE_wol_disable(&g_QSE);
        
        return 0;
    }
```

</div>


 -------------------------------- 
<a name="QSE_get_tx_stats"></a>
## QSE_get_tx_stats
<a name="prototype"></a>
### Prototype 

<div id="Functions$QSE_get_tx_stats$prototype" data-type="code">

    void
    QSE_get_tx_stats
    (
        QSE_instance_t * this_QSE,
        QSE_tx_stats_t * stats_out
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$QSE_get_tx_stats$description" data-type="text">

The QSE_get_tx_stats() function reads all enabled transmit statistics counters.
It populates the QSE_tx_stats_t structure with current counter values.
Counters not enabled in the core configurator return zero value. Each counter 
clears to zero in hardware upon being read (read-to-clear behavior).

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_QSE
<div id="Functions$QSE_get_tx_stats$description$parameters$this_QSE" data-type="text" data-name="this_QSE">

The this_QSE parameter points to a QSE_instance_t structure that holds 
all data related to the CoreQSE instance.


</div>

#### stats_out
<div id="Functions$QSE_get_tx_stats$description$parameters$stats_out" data-type="text" data-name="stats_out">

The stats_out parameter points to a QSE_tx_stats_t structure to receive
the transmit statistics counter values.


</div>

<a name="return"></a>
### Return
<div id="Functions$QSE_get_tx_stats$description$return" data-type="text">

This function does not return a value.


</div>

##### Example 
<div id="Functions$QSE_get_tx_stats$description$example$Example1" data-type="text" data-name="Example">

</div>

<div id="Functions$QSE_get_tx_stats$description$example$Example1" data-type="code" data-name="Example">


```
    QSE_instance_t g_QSE;
    QSE_tx_stats_t tx_stats;
    
    void print_tx_statistics(void)
    {
        /* Read transmit statistics (clears counters) */
        QSE_get_tx_stats(&g_QSE, &tx_stats);
        
        /* Process statistics as needed */
        /* tx_stats.tx_frame_count, tx_stats.tx_byte_count, etc. */
    }
```

</div>


 -------------------------------- 
<a name="QSE_get_rx_stats"></a>
## QSE_get_rx_stats
<a name="prototype"></a>
### Prototype 

<div id="Functions$QSE_get_rx_stats$prototype" data-type="code">

    void
    QSE_get_rx_stats
    (
        QSE_instance_t * this_QSE,
        QSE_rx_stats_t * stats_out
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$QSE_get_rx_stats$description" data-type="text">

The QSE_get_rx_stats() function reads all enabled receive statistics counters.
It populates the QSE_rx_stats_t structure with current counter values.
Counters not enabled in the core configurator return zero. Each counter clears
to zero in hardware upon being read (read-to-clear behavior).

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_QSE
<div id="Functions$QSE_get_rx_stats$description$parameters$this_QSE" data-type="text" data-name="this_QSE">

The this_QSE parameter points to a QSE_instance_t structure that holds 
all data related to the CoreQSE instance.


</div>

#### stats_out
<div id="Functions$QSE_get_rx_stats$description$parameters$stats_out" data-type="text" data-name="stats_out">

The stats_out parameter points to a QSE_rx_stats_t structure to receive
the receive statistics counter values.


</div>

<a name="return"></a>
### Return
<div id="Functions$QSE_get_rx_stats$description$return" data-type="text">

This function does not return a value.


</div>

##### Example 
<div id="Functions$QSE_get_rx_stats$description$example$Example1" data-type="text" data-name="Example">

</div>

<div id="Functions$QSE_get_rx_stats$description$example$Example1" data-type="code" data-name="Example">


```
    QSE_instance_t g_QSE;
    QSE_rx_stats_t rx_stats;
    
    void print_rx_statistics(void)
    {
        /* Read receive statistics (clears counters) */
        QSE_get_rx_stats(&g_QSE, &rx_stats);
        
        /* Process statistics as needed */
        /* rx_stats.rx_frame_count, rx_stats.rx_byte_count, etc. */
    }
```

</div>


 -------------------------------- 
<a name="QSE_clear_stats"></a>
## QSE_clear_stats
<a name="prototype"></a>
### Prototype 

<div id="Functions$QSE_clear_stats$prototype" data-type="code">

    void
    QSE_clear_stats
    (
        QSE_instance_t * this_QSE
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$QSE_clear_stats$description" data-type="text">

The QSE_clear_stats() function clears all enabled transmit and receive 
statistics counters. It clears counters by reading each active counter register
and discarding the value, utilizing the hardware read-to-clear behavior.
After this call, all active statistics counters are reset to zero.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_QSE
<div id="Functions$QSE_clear_stats$description$parameters$this_QSE" data-type="text" data-name="this_QSE">

The this_QSE parameter points to a QSE_instance_t structure that holds 
all data related to the CoreQSE instance.


</div>

<a name="return"></a>
### Return
<div id="Functions$QSE_clear_stats$description$return" data-type="text">

This function does not return a value.


</div>

##### Example 
<div id="Functions$QSE_clear_stats$description$example$Example1" data-type="text" data-name="Example">

</div>

<div id="Functions$QSE_clear_stats$description$example$Example1" data-type="code" data-name="Example">


```
    QSE_instance_t g_QSE;
    
    void reset_statistics(void)
    {
        /* Clear all statistics counters */
        QSE_clear_stats(&g_QSE);
    }
```

</div>


 -------------------------------- 
<a name="QSE_pcs_get_sync_status"></a>
## QSE_pcs_get_sync_status
<a name="prototype"></a>
### Prototype 

<div id="Functions$QSE_pcs_get_sync_status$prototype" data-type="code">

    uint8_t
    QSE_pcs_get_sync_status
    (
        QSE_instance_t * this_QSE
    );


</div>

<a name="description"></a>
### Description

<div id="Functions$QSE_pcs_get_sync_status$description" data-type="text">

The QSE_pcs_get_sync_status() function returns the synchronization state of the
Physical Coding Sublayer (PCS). It reads the SYNC status driven by the PCS
synchronization state machine.

</div>


 --------------------------- 

<a name="parameters"></a>
### Parameters
#### this_QSE
<div id="Functions$QSE_pcs_get_sync_status$description$parameters$this_QSE" data-type="text" data-name="this_QSE">

The this_QSE parameter points to a QSE_instance_t structure that holds 
all data related to the CoreQSE instance.


</div>

<a name="return"></a>
### Return
<div id="Functions$QSE_pcs_get_sync_status$description$return" data-type="text">

This function returns:
- 1 when PCS has achieved a valid lock state and receive link is active
- 0 when PCS is not synchronized


</div>

##### Example 
<div id="Functions$QSE_pcs_get_sync_status$description$example$Example1" data-type="text" data-name="Example">

</div>

<div id="Functions$QSE_pcs_get_sync_status$description$example$Example1" data-type="code" data-name="Example">


```
    QSE_instance_t g_QSE;
    
    int main(void)
    {
        QSE_init(&g_QSE, CoreQSE_BASE_ADDR);
        QSE_tx_enable(&g_QSE);
        QSE_rx_enable(&g_QSE);
        
        /* Wait for PCS synchronization */
        while (QSE_pcs_get_sync_status(&g_QSE) == 0)
        {
            /* Wait for link */
        }
        
        /* Link is now active, proceed with data transfer */
        
        return 0;
    }
```

</div>


 -------------------------------- 
