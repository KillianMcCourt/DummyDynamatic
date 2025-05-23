# Timing Information and its handling in Dynamatic

This document aims to present all essential information relating to the component timing information, in order to document existing code and ease future implementations which would affect it.

# High-level overview

## What timing information is meant here & its importance

In order to realise a hardware circuit capable of functioning correctly, it is necessary to know to know, for every operator, their **latency** and their **delay**:

 - **latency**  is the number of cycles the operator requires to output a result from provided data. It is always an integer. This is essential to ensure data synchronisation across the datapaths.
  
 - **delay** is generally the time (indicated in μs) taken for the operator to compute its result from the moment the input data becomes available to the moment the output data is produced. In our case, we distinguish three useful forms of delay : 
    -operator delay, which is delay not comprised between internal registers, which must be accounted for when computing the slack of a path including the operator. Worth 0 for most operators with a latency greater than 0, due to these generally ending with a register.

    -handshake delays, which account for the delay before a signal is respectively valid and ready to be passed through the transaction.
    
    -internal combinational delay, which is the highest delay between two internal registers, which defines the maximum frequency supported by this operator. Not defined for 0 latency operators. Not currently supported in Dynamatic.


## How this information is stored

All timing information relative to the operators is stored in the [components JSON]((https://github.com/EPFL-LAP/dynamatic/blob/main/data/components.json). A sample element is listed as

``` "handshake.addi": {
    "latency": {
      "64": 0.0
    },
    "delay": {
      "data": {
        "1": 1.397,
        "2": 1.397,
        "4": 2.038,
        "8": 1.927,
        "16": 2.047,
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
    "inport": {
      "delay": {
        "data": {
          "64": 0
        },
        "valid": {
          "1": 0
        },
        "ready": {
          "1": 0
        },
        "VR": 0,
        "CV": 0,
        "CR": 0,
        "VC": 0,
        "VD": 0
      }
    },
    "outport": {
      "delay": {
        "data": {
          "64": 0
        },
        "valid": {
          "1": 0
        },
        "ready": {
          "1": 0
        },
        "VR": 0,
        "CV": 0,
        "CR": 0,
        "VC": 0,
        "VD": 0
      }
    }
  }*
```


The latency is listed as a function of the (integer) bitwidth.
The operator delay is also given as a function of bitwidth. Valid and ready delays are listed seperately, as are the signal arrival order dependency delays. Delay information for inports and outports is in a seperate section, with the same organisation.


## How this information is used

Timing information is currently primarily required during the buffer placement stage, which ensures the paths through the system are correct. The basic "on-merge" buffer placement algorithm is not timing-aware, and does not make use of it; but both supported MILP algorithms (fpga20 and flp22) do. For this, the information is extracted from the JSON, and stored into dedicated timing structures, from which they will be read. These are covered in detail in the following section.

# Implementation 

## Timing Structures

Consider the following representation : 

![image](https://github.com/user-attachments/assets/50d75d46-ea0e-4300-a354-60aaa7ed8754)

During a pass requiring timing data, such as buffer placement, a [TimingDatabase](https://github.com/EPFL-LAP/dynamatic/blob/main/include/dynamatic/Support/TimingModels.h#L174) is instanciated. This is the top-level class, which contains all timing data of the entire IR. It is initially empty and is then filled (see next section) with instances of the [TimingModel](https://github.com/EPFL-LAP/dynamatic/blob/main/include/dynamatic/Support/TimingModels.h#L103) struct. Each instance of this struct is associated to a single Op of the IR, and contains all the data relative to this Op. 

## JSON Deserialisation



