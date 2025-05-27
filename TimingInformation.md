# Timing Information and its handling in Dynamatic

This document explains how Dynamatic stores and uses timing information for hardware operators, providing both conceptual understanding and implementation guidance.

## What is Timing Information?

For hardware circuits to function correctly, every operator needs two key timing properties:

- **Latency**: The number of clock cycles an operator takes to produce output from input data (always an integer). Essential for keeping data synchronized across different paths.

- **Delay**: The actual time (in microseconds) for an operator to compute its result. We track several types:
  - Operator delay: Processing time not including internal registers
  - Handshake delays: Time for valid/ready signals in transactions  
  - Internal combinational delay: Maximum delay between internal registers (defines max frequency)

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
      "CV": 0
    },
    "inport": { /* port-specific delays, structured like the delay set above */ },
    "outport": { /* port-specific delays */ }
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
