# Introduction

This document specifies the implementation of the *CPU Debug Unit*. The CPU Debug Unit contains some Event Monitors, Snapshot Collectors, one Snapshot Data Correlation Module (SDCM) and one Packetizer.

The event monitor detects certain events and informs the SDCM about their occurrences. At the moment three different event monitors are implemented: the Program Counter Monitor, the Function Return Monitor and the Memory Address Monitor. The Program Counter Monitor compares the current value of the Program Counter with pre-defined events. If there is a match the module informs the SDCM.
The Function Return Monitor detects the return of a certain function by means of the program counter and the return address which is stored in the CPU registers when the function is called. The Memory Address Monitor observes the memory writes of the CPU and compares it to the preconfigured memory address singal values. Like at the other monitors this module forwards an event signal in case of a match.

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
 `memaddr_val`      | System->CPU Debug Unit | Memory Interface (of Memory Address Monitor), address valid
 `sram_ce`          | System->CPU Debug Unit | Memory Inferface (of Memory Address Monitor), chip enable
 `sram_we`          | System->CPU Debug Unit | Memory Interface (of Memory Address Monitor), write enable
 `pc_val`	    | System->CPU Debug Unit | Program Counter Interface (of Program Counter Monitor), Program Counter valid
 `pc_enable`        | System->CPU Debug Unit | Program Counter Interface (of Program Counter Monitor), Program Counter enable
 `wb_enable`        | System->CPU Debug Unit | Writeback Register Interface (of Function Return Monitor), writeback enable
 `wb_reg`           | System->CPU Debug Unit | Writeback Register Interface (of Function Return Monitor), writeback register
 `wb_data`          | System->CPU Debug Unit | Writeback Register Interface (of Function Return Monitor), writeback data
 `trace_insn`       | System->CPU Debug Unit | Instruction Trace Interface (of the Stack), trace insn
 `trace_enable`     | System->CPU Debug Unit | Instruction Trace Interface (of the Stack), trace enable

# Memory Map

 Address Range | Description
 ------------- | -----------
 `0x200`       | Address width in Byte
 `0x201`       | Data width in Byte
 `0x202`       | `1` if unaligned accesses are allowed, `0` otherwise

# Debug Content

Before an event can be triggered the event has to be described.
This can be done with the Debug Content Registers.
The structure of the registers is described below:

 Index   | Content                          | Remark
 ------- | -------                          | ------
 0x210   | `ON/OFF`			    | Bit 0 is used for ON/OFF
 0x211   | `PC Config Event_1 1/3`	    |
 0x212   | `PC Config Event_1 2/3`	    | Description......
 0x213   | `PC Config Event_1 3/3`	    |
 0x214   | `PC Config Event_2 1/3`	    |
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