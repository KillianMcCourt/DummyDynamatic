# Summary

This PR corresponds to the described changes as PR1 of issue [#457](https://github.com/EPFL-LAP/dynamatic/issues/457). The objective is to lay groundwork necessary to support units that vary in both latency and internal operating frequency - since this frequency is defined by the internal combinational delay, we will focus on the handling of this "delay" going forwards. See the following figure for an illustration of the difference between this delay and the already-supported operator-wide delay : 

![Image](https://github.com/user-attachments/assets/7aa06237-ca44-48b9-90a9-870ec21e94dd)

This first PR focuses on changes to the implementation of the JSON and the latency extraction, but leaves the IR and the backend unit generation untouched.

# Motivation

The current Dynamatic HLS tool assumes each pipelined operation has a single, fixed-latency implementation.

This limitation is built in three places of the timing information:

- **components.json**
  - Only lists `bitwidth:latency` pairs, without dependency on delay.

- **Parsing logic**
  - Expects the latency field to be a list of such pairs.

- **TimingModel**
  - Built to store this structure.
  - Getter logic depends on this structure.

This means the current system has no way of:

- Storing internal combinational delays.
- Returning the correct latency associated with such a delay.

This matters because:

- The maximum frequency of a circuit is limited by the slowest multi-cycle operator.
- Speed-area tradeoffs along the Pareto frontier are standard in FPGA design.


# Assumptions 

This PR makes 2 assumptions :

  - that any further components added be added with the new format (see below) - a placeholder delay of 0.0 can be used
  - that any TimingDatabase::getLatency call will now provide TargetCP as an argument

# Implementation


## Expanding  ```components.json```  



In order to expand the structure of ```components.json``` representing latency, we have to understand the current structure.



The current structure for latency value is the following:



```



  "MLIR_OP": {

    "latency": {

      "BITWIDTH": LATENCY_VALUE

    },



```



where ```MLIR_OP``` is the name fo the MLIR operation, ```BITWIDTH``` correponds to the bitwidth of the operation, and ```LATENCY_VALUE``` is the value of the latency for the specific MLIR operation with the specific bitwidth value.


Therefore, our approach will update the latency field to a map of maps, where the bitwidth will be the keys of the outer map; and delays keys of the inner map, with latency values being the data. This yields :



```



  "MLIR_OP": {

    "latency": {

      "BITWIDTH": {

        "INTERNAL_COMB_DELAY_1": LATENCY_VALUE_1,

        "INTERNAL_COMB_DELAY_2": LATENCY_VALUE_2

      },

    },




```



where ```INTERNAL_COMB_DELAY_1``` and ```INTERNAL_COMB_DELAY_2``` represent the maximum internal combinational delay between two registers of two different implementation of the same MLIR operation with the same bitwidth. These values correspond to the delay of the combinational logic of the figure. 

**Please note that for now, the delay values will be chosen realistically, but the latencies listed will be fixed at those of the current ilmplementation in backend, to avoid inducing performance reductions. This will be replaced with the correct values in an upcoming backend PR.**

### Adding a new data structure


The current JSON file uses bitwidth values as keys to distinguish between different implementations of the same MLIR operation. This design choice closely reflects the underlying data structure used to store this information. Specifically, for each distinct timing entry, the ```TimingModel``` contains a [```BitwidthDepMetric```](https://github.com/EPFL-LAP/dynamatic/blob/main/include/dynamatic/Support/TimingModels.h#L46) structure. This structure organizes the data as a map, where the keys are unsigned integers representing bitwidth values and the values are the different timing information (i.e., latency, operator-wide delay). In the code, [```latency```](https://github.com/EPFL-LAP/dynamatic/blob/main/include/dynamatic/Support/TimingModels.h#L115) and [```dataDelay```](https://github.com/EPFL-LAP/dynamatic/blob/main/include/dynamatic/Support/TimingModels.h#L117) are two examples. However, ```BitwidthDepMetric``` is generic enough to allow any possible value type. For this reason, we re-use the same data structure for our implementation.




In our proposal, the ```BitwidthDepMetric``` latency object stores another map as the value for each key, aligning with the structure of the JSON file. To represent this nested map, we introduce a new data structure called ```CombDelayDepMetric```, which uses combinational delay values as keys.



The main purpose of defining ```CombDelayDepMetric``` is to replicate the behavior of ```BitwidthDepMetric```. Both structures are essentially maps constrained to a fixed key type. A key advantage of ```BitwidthDepMetric``` is its getter function, [```getCeilMetric```](https://github.com/EPFL-LAP/dynamatic/blob/main/include/dynamatic/Support/TimingModels.h#L58), which accepts a bitwidth value and returns the value associated with the closest key if an exact match is not found.



Similarly, we plan to implement a corresponding getter for ```CombDelayDepMetric```, named ```getCombDelayCeilMetric```.







### Parsing the updated '''components.json'''  



The following is the pseudo-code of the parsing process:

Note that the variable ```value```  in the pseudo code corresponds to one of the timing information field (for instance, double or BitwidthDepMetric object).

```

  TimingDatabase db;

  components = JSON.get_list_components()

  

  For each component in components:

    TimingModel model;

  

    For each key in KEYS:

      deserializeNested( key, value );

      addValue( model, key, value );



    db.add( component, model );

```



The current flow starts by instantiating an empty timingDatabase; for every component, it then defines a TimingModel object, which contains fields for relevant information, such a ```BitwidthDepMetric<double> latency>. ```.

Each timing model's data fields are then filled one by one,  by deserialising the relevant nested information from ```components.json```. 

Deserialisation makes use of a function ```fromJSON```. This function takes a reference to the field, and writes the data back into it. Since several field types exist, it has several overloads (currently, for double ( and [BitwdithDepMetric<double>](https://github.com/EPFL-LAP/dynamatic/blob/main/lib/Support/TimingModels.cpp#L283) ).

Since our change involves changing the type of the latency field to a new one (BitwdithDepMetric<CombDelayDepMetric<double>>) , we add an overload to handle it - it will instantiate a DelayDepmetric struct for every bitwidth, and store the corresponding map into the struct's data field.

Pseudo-code for this overload is :

```

For each (bitwidthKey, metricValue) in JSON object:
  Parse bitwidthKey → bitwidth
  Create empty LatencyMap

  For each (doubleKey, doubleValue) in nested metricValue object:
    Parse doubleKey → delay
    Parse doubleValue → latency
    LatencyMap[delay] = latency

  Create DelayDepMetric with LatencyMap
  metric[bitwidth] = DelayDepMetric

```





### Selecting the operation latency



The main objective  of this PR is to allow the various algorithms to obtain the latency corresponding to a chosen implementation. 

The selection logic works as follows: given the target operating frequency, we find the best matching latency by running two functions sequentially. First, getCeilMetric extracts the appropriate delay:latency map based on bitwidth requirements. Then, CombDelayDepMetric selects the optimal latency from this map based on combinational delay constraints.
This approach ensures consistent latency choices across all getLatency calls without tying the selection to any specific part of the run.

![image](https://github.com/user-attachments/assets/472fb356-0cf5-4735-abf8-b1251d8e33a9)







# Full example :


Let us consider the case of the floating point adder, here on referred to as addf.


## JSON values

Current value is : 

```



  "handshake.addf": {

    "latency": {

      "64": 9.0

    },

```

the updated structure for ```addf``` is the following:



```



  "handshake.addf": {

    "latency": {

      "64": {

        "4.1" : 9.0,
        "5.0" : 9.0
      },

    },

```




Note that as mentionned above, I list the same latency for all delays, in order to ensure consistency with the unaltered backend. 

