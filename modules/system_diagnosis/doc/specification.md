# Introduction

This document specifies the implementation of the *CPU Debug Unit*. The CPU Debug Unit contains some Event Monitors, Snapshot Collectors, one Snapshot Data Correlation Module (SDCM) and one Packetizer.

The event monitor detects certain events and informs the SDCM about their occurrences. At the moment two different event monitors are implemented: the Program Counter Monitor and the Function Return Monitor. The Program Counter Monitor compares the current value of the Program Counter with pre-defined events. If there is a match the module informs the SDCM.
The Function Return Monitor detects the return of a certain function by means of the program counter and the return address which is stored in the CPU registers when the function is called.

When an event occurs the SDCM gets informed by the event monitors. Depending on the event configuration the SDCM is responsible to trigger the Snapshot Collectors in order to collect the requested data.

The packetizer generates trace packets containing the Event-ID, a timestamp and the requestet data from the Snapshot Collectors. Additionally, the packetizer is the interface to the Debug Co-Processor and the Host. It is responsible to forward the trace packets and it receives the configuration of the events. 



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
 `memaddr_val`      | System->CPU Debug Unit | Memory Interface, address valid
 `sram_ce`          | System->CPU Debug Unit | Memory Inferface, chip enable
 `sram_we`          | System->CPU Debug Unit | Memory Interface, write enable
 `time_global`      | System->CPU Debug Unit | Interface to the global timestamp
 `traceport_flat`   | System->CPU Debug Unit | Execution traceport in a single signal<br>pc_val, pc_enable, wb_enable, wb_reg,<br>wb_data, trace_isn, trace_enable
 `dbgnoc_in_flit`   | System->CPU Debug Unit | Debug NoC Interface, input data
 `dbgnoc_in_valid`  | System->CPU Debug Unit | Debug NoC Interface, input valid
 `dbgnoc_out_ready` | System->CPU Debug Unit | Debug NoC Interface, output ready
 `dbgnoc_out_flit`  | CPU Debug Unit->System | Debug NoC Interface, output data
 `dbgnoc_out_valid` | CPU Debug Unit->System | Debug NoC Interface, output valid
 `dbgnoc_in_ready`  | CPU Debug Unit->System | Debug NoC Interface, input ready


# Debug Content

Before an event can be triggered the event has to be described.
This can be done with the Debug Content Registers.
The structure of the registers is described below:

 Index   | Content                          | Remark
 ------- | -------                          | ------
 0x00    | `MODULE_TYPE	MODULE_VERSION`     | Bit 0-7 MODULE_VERSION, Bit 8-15 MODULE_TYPE
 0x01    | `CORE_ID`			    |
 0x02    | `ON/OFF`			    | Bit 0 is used for ON/OFF
 0x03    | `PC Config Event_1 1/3`	    |
 0x04    | `PC Config Event_1 2/3`	    |
 0x05    | `PC Config Event_1 3/3`	    |
 0x06    | `PC Config Event_2 1/3`	    |
 ..      | `..`				    |
         | `Fcn Return Config Event_1 1/3`  |
         | `..`				    |


# Trace Packets

The trace packets were sent over the Debug NoC to the host.
The sequence starts with a common header:

 Type     | Content       						      | Remark
 -------  | -------							      | ------
 Header   | `[15]`: R/W, `[14]`: Single/Chunk, `[13:0]`: Strobe/Chunk size    |
 Content  | `EVENT ID`							      | Bit 0-5 Event-ID
 Content  | `TIMESTAMP_LSB`						      |
 Content  | `TIMESTAMP_MSB`						      |
 Content  | `GPR DATA LSB`						      |
 Content  | `GPR DATA MSB`						      |
 ..       | ..								      |
 Content  | `STACKARG DATA LSB`						      |
 Content  | `STACKARG DATA MSB`						      |
 Tail     | `Address[15:0]`						      |