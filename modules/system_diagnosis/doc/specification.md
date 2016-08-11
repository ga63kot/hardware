# Introduction

This document specifies the implementation of the *CPU Debug Unit*. The module contains some Event Monitors, Snapshot Collectors, one Snapshot Data Correlation Module (SDCM) and one Packetizer.

The event monitor detects certain events and informs the SDCM about their occurances. At the moment two different event monitors are included: the Program Counter Monitor and the Function Return Monitor. The Program Counter Monitor compares the current value of the Program Counter with pre-defined events. If there is a match the module informs the SDCM.
The Function Return Monitor detects the return of a certain function by means of the program counter and the return address which is stored in the CPU registers when the function is called.

The SDCM gets informed by the event monitors when an event occurs. Depending on the event configuration the SDCM is responsible to inform the Snapshot collectors in order to collect the data which is correlated with the event.

The packetizer generates trace packets which contains the Event-ID, a timestamp and the requestet datas from the Snapshot Collectors. Additionally, the packetizer is the interface to the Debug Co-Processor and the Host. It is responsible to forward the trace packets and it receives the configuration of the events. 



## License

This work is licensed under the Creative Commons
Attribution-ShareAlike 4.0 International License. To view a copy of
this license, visit
[http://creativecommons.org/licenses/by-sa/4.0/](http://creativecommons.org/licenses/by-sa/4.0/)
or send a letter to Creative Commons, PO Box 1866, Mountain View, CA
94042, USA.

You are free to share and adapt this work for any purpose as long as
you follow the following terms: (i) Attribution: You must give
appropriate credit and indicate if changes were made, (ii) ShareAlike:
If you modify or derive from this work, you must distribute it under
the same license as the original.

## Authors

Tim Fritzmann, Markus Göhrle

# System Interface

There is a generic interface between the CPU Debug Unit and the system:

 Signal             | Direction              | Description
 -------------------| -----------------------| -----------
 `clk`              | System->CPU Debug Unit | System CPU Clock
 `reset`            | System->CPU Debug Unit | System Reset
 `memaddr_val`      | System->CPU Debug Unit | Memory Interface
 `sram_ce`          | System->CPU Debug Unit | Memory Inferface, Chip Enable
 `sram_we`          | System->CPU Debug Unit | Memory Interface, Write Enable
 `time_global`      | System->CPU Debug Unit | Interface to the global timestamp
 `traceport_flat`   | System->CPU Debug Unit | Execution traceport in a single signal
 `dbgnoc_in_flit`   | System->CPU Debug Unit | Debug NoC Interface, input data
 `dbgnoc_in_valid`  | System->CPU Debug Unit | Debug NoC Interface, input valid
 `dbgnoc_out_ready` | System->CPU Debug Unit | Debug NoC Interface, output ready
 `dbgnoc_out_flit`  | CPU Debug Unit->System | Debug NoC Interface, output data
 `dbgnoc_out_valid` | CPU Debug Unit->System | Debug NoC Interface, output valid
 `dbgnoc_in_ready`  | CPU Debug Unit->System | Debug NoC Interface, input ready

# Memory Map

 Address Range | Description
 ------------- | -----------
 `0x200`       | Address width in Byte
 `0x201`       | Data width in Byte
 `0x202`       | `1` if unaligned accesses are allowed, `0` otherwise

# Debug Content

Before an event can be triggered the event has to be described.
This can be done with the Debug Content Registers.
The Content of the registers are described below:

 Index   | Content
 ------- | -------
 0       | `MODULE_TYPE	MODULE_VERSION`
 1       | `CORE_ID`
 2       | `ON/OFF`
 3       | `PC Config Event_1 1/3`
 4       | `PC Config Event_1 2/3`
 5       | `PC Config Event_1 3/3`
 5       | `PC Config Event_2 1/3`
 ..      | `..`
         | `Fcn Return Config Event_1 1/3`
         | `..`


# Trace Packets

The trace packets were sent over the Debug NoC to the host.
The sequence starts with a common header:

 Payload | Content
 ------- | -------
 0       | `[15]`: R/W, `[14]`: Single/Chunk, `[13:0]`: Strobe/Chunk size
 1       | `EVENT ID`
 2       | `TIMESTAMP_LSB`
 3       | `TIMESTAMP_MSB`
 4       | `GPR DATA LSB`
 5       | `GPR DATA MSB`
 ..      | ..
 N-2     | `STACKARG DATA LSB`
 N-1     | `STACKARG DATA MSB`
 N       | `Address[15:0]`

Following this setup information, the actual transfer takes place be
sending the data in a stream, meaning the debug packets are maximally
filled and the data spans multiple packets. It is only possible to
have one read or one write access at a time.
    (dest=MAM_ID)
	(type=PLAIN,src=0)
	D3[15:0]

