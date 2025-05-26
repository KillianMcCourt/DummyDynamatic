# Timing Information in Dynamatic

## Overview

Dynamatic needs timing information to generate correct hardware circuits. This document explains how timing data is stored, loaded, and used throughout the system.

## What is Timing Information?

Every hardware operator needs two key timing properties:

- **Latency**: Number of clock cycles required to produce output (always an integer)
- **Delay**: Time in microseconds for signal propagation through the operator

These values ensure data synchronization across datapaths and enable proper buffer placement algorithms.

## Types of Delays

Dynamatic tracks three types of delays:

1. **Operator delay**: Propagation time through combinational logic (not between registers)
2. **Handshake delays**: Time for valid/ready signals to propagate 
3. **Internal combinational delay**: Maximum delay between internal registers (defines max frequency)

## Data Storage

All timing information is stored in `data/components.json`. Here's a simplified example:

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
      }
}
```

Values are organized by bitwidth since timing often depends on data width.

## How It Works

### Loading Process

1. **Initialization**: Create a `TimingDatabase` object
2. **JSON Reading**: Parse the components.json file
3. **Model Creation**: Build `TimingModel` objects for each operator type
4. **Database Population**: Store models in the database for runtime access

### Runtime Usage

The system provides several getter functions:

- `getLatency()`: Returns cycle count for an operation
- `getInternalDelay()`: Returns operator's internal propagation delay  
- `getPortDelay()`: Returns input/output port delays
- `getTotalDelay()`: Returns complete delay including ports

### Key Classes

- **TimingDatabase**: Top-level container holding all timing data
- **TimingModel**: Per-operator timing information 
- **BitwidthDepMetric**: Handles bitwidth-dependent values
- **PortModel**: Stores input/output port timing

## Usage in Practice

Timing information is primarily used during **buffer placement**, where algorithms need to:

- Calculate path delays through the circuit
- Ensure proper data synchronization 
- Place buffers to meet timing constraints

The basic buffer placement algorithm ignores timing, but advanced MILP algorithms (fpga20, flp22) rely heavily on this data.

## Implementation Notes

### Bitwidth Handling

Many timing values depend on data width. The `BitwidthDepMetric` class automatically selects the appropriate value using `getCeilMetric()`, which finds the smallest available bitwidth greater than or equal to the requested size.

### Special Cases

- **LSQ Operations**: Add 3 extra cycles to account for additional load latency
- **Zero-latency operators**: Only have combinational delays, no register-to-register timing
- **Missing data**: System gracefully handles operators not in the JSON

### Error Handling

The loading process includes comprehensive error checking:
- JSON parsing failures
- Missing required fields  
- Invalid data formats
- File access issues

## Extending the System

To add timing data for new operators:

1. Add entry to `components.json` following the existing format
2. Include latency and delay values for relevant bitwidths
3. Specify inport/outport delays if different from operator delays
4. Test with buffer placement algorithms to verify correctness

The system automatically picks up new entries without code changes.
