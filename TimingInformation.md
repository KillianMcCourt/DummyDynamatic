# Timing Information and its handling in Dynamatic

This document explains how Dynamatic stores and uses timing information for hardware operators, providing both conceptual understanding and implementation guidance.

## What is Timing Information?

For hardware circuits to function correctly, every operator needs two key timing properties:

- **Latency**: The number of clock cycles an operator takes to produce output from input data (always an integer). Essential for keeping data synchronized across different paths.

- **Delay**: The actual time (in microseconds) between the arrival of a signal, and the availability of the corresponding output. We track several types:
  - **Single signal type delays**: Time taken within a port/component to compute the output. We consider data delays within this case, as well as the "ready" and "valid" delays internal to a port/component.
  - **Cross signal type delays**: Time from the propagation of handshaking signals from one signal to another of a different type. five of these are supported : 
    - VR ( valid to ready), CV (control to valid), CR (control to ready), VC (valid to control), VD (valid to data)

 
  **An important point** is that there is a major distinction between **pipelined** and **non-pipelined** operations. Consider the following graph :

  ![image](https://github.com/user-attachments/assets/5fa5df02-ac2e-4930-a88e-137c821716a9)


In the pipelined (or latency > 0) case, for any signal (data or handshaking like valid, ready, control), an inport and outport are used, but the internal delay is covered by the registers; therefore the delay to be accounted for is that of the ports.

In the non-pipelined case, no ports are needed, and the delays to be accounted for are those of the operator itself.

Note : This documentation is based on our understanding of the codebase, however we are not its original authors and can only speculate on the intended usage of the ports. This is because the current code does not seem actually use the data parsed from the storage that relates to the port models; and furthermore all the port delays are 0 for all listed components. 




## Where Timing Data is Stored

All timing information lives in the [components JSON file](https://github.com/EPFL-LAP/dynamatic/blob/main/data/components.json). Here's what a typical entry looks like:

```json
{
  "handshake.addi": {
    "latency": {
      "64": 0.0
    },
    "delay": {
      "data": {
        "32": 2.287,
        "64": 2.767
      },
      "valid": {
        "1": 1.397
      },
      "ready": {
        "1": 1.4
      },
      "VR": 1.409,
      "CV": 0,
      "CR": 0,
      "VC": 0,
      "VD": 0
    },
    "inport": { /* port-specific delays, structured like the delay set above */ },
    "outport": { /* port-specific delays, structured like the delay set above  */ }
  }
}
```

The structure captures:
- Latency values by bitwidth
- Processing delays by bitwidth and signal type
- Signal ordering dependencies (VR, CV, etc.)
- Separate timing for input/output ports

## How Timing Information is Used

Timing data is primarily used during **buffer placement**, which ensures correct dataflow timing. While basic buffer placement ignores timing, the advanced MILP algorithms (fpga20 and flp22) rely heavily on this information to optimize circuit performance.

# Implementation Overview

## Core Data Structures

The timing system uses a two-level hierarchy:

**[TimingDatabase](https://github.com/EPFL-LAP/dynamatic/blob/main/include/dynamatic/Support/TimingModels.h#L174)**: Top-level container holding timing data for the entire IR
- Stores multiple TimingModel instances
- Provides getter methods to extract timing information
- Gets populated from the JSON file during buffer placement passes

**[TimingModel](https://github.com/EPFL-LAP/dynamatic/blob/main/include/dynamatic/Support/TimingModels.h#L103)**: Contains all timing data for a single operation
- Stores latency, delays, and port information
- Uses BitwidthDepMetric for bitwidth-dependent values
- Contains nested PortModel structures for input/output timing

- TimingModels must handle bitwidth-dependent data. For this, a specific struc is used :

**[BitwidthDepMetric](https://github.com/EPFL-LAP/dynamatic/blob/main/include/dynamatic/Support/TimingModels.h#L46)**: Handles bitwidth-dependent timing values
- Maps bitwidths to latency values (e.g., 32-bit → 9 cycles)
- Provides getCeilMetric() to find the right value for a given bitwidth

## Loading Timing Data from JSON

The process follows these steps:

1. **Initialization**: Create empty TimingDatabase
2. **File Reading**: Parse the entire components.json file
3. **Data Extraction**: For each operator in the JSON:
   - Create a TimingModel instance
   - Extract latency, delay, and port timing data, with appropriate calls to an overloaded fromJSON function (example for BitwidthDepMetric [here](https://github.com/EPFL-LAP/dynamatic/blob/main/lib/Support/TimingModels.cpp#L373) )
   - Handle bitwidth-dependent values appropriately
   - Insert the completed model into the TimingDatabase

The JSON parsing handles the nested structure automatically, converting string keys to bitwidths and organizing delay values by signal type.

## Accessing Timing Information

The TimingDatabase provides several getter methods:

- **[getLatency()](https://github.com/EPFL-LAP/dynamatic/blob/main/lib/Support/TimingModels.cpp#L114)**: Returns cycle count for an operation
- **[getInternalDelay()](https://github.com/EPFL-LAP/dynamatic/blob/main/lib/Support/TimingModels.cpp#L143)**: Gets processing delay excluding ports
- **[getPortDelay()](https://github.com/EPFL-LAP/dynamatic/blob/main/lib/Support/TimingModels.cpp#L161)**: Gets delay for specific input/output ports  
- **[getTotalDelay()](https://github.com/EPFL-LAP/dynamatic/blob/main/lib/Support/TimingModels.cpp#L183)**: Sums all relevant delays for an operation path

These methods automatically handle bitwidth lookup and return the appropriate timing value for the requested operation and signal type.


---

This timing system ensures Dynamatic can generate hardware that meets timing requirements while providing the flexibility to optimize for different bitwidths and operation types.
