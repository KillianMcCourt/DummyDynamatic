th# Timing Information and its handling in Dynamatic

This document aims to present all essential information relating to the component timing information, in order to document existing code and ease future implementations which would affect it. For concision and readability, pseudo-code is used throughout - links to the implementations are systematically provided for the advanced reader.

# High-level overview

## What timing information is meant here & its importance

In order to realise a hardware circuit capable of functioning correctly, it is necessary to know to know, for every operator, their **latency** and their **delay**:

 - **latency**  is the number of cycles the operator requires to output a result from provided data. It is always an integer. This is essential to ensure data synchronisation across the datapaths.
  
 - **delay** is generally the time (indicated in μs) taken for the operator to compute its result from the moment the input data becomes available to the moment the output data is produced. In our case, we distinguish three useful forms of delay : 
    -operator delay, which is delay not comprised between internal registers, which must be accounted for when computing the slack of a path including the operator. Worth 0 for most operators with a latency greater than 0, due to these generally ending with a register.

    -handshake delays, which account for the delay before a signal is respectively valid and ready to be passed through the transaction.
    
    -internal combinational delay, which is the highest delay between two internal registers, which defines the maximum frequency supported by this operator. Not defined for 0 latency operators. Not currently supported in Dynamatic.


## How this information is stored

All timing information relative to the operators used in Dynamatic is stored in the [components JSON](https://github.com/EPFL-LAP/dynamatic/blob/main/data/components.json). A sample element is listed as

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

### Storage

Consider the following representation of the stored data : 

![image](https://github.com/user-attachments/assets/50d75d46-ea0e-4300-a354-60aaa7ed8754)

During a pass requiring timing data, such as buffer placement, a [TimingDatabase](https://github.com/EPFL-LAP/dynamatic/blob/main/include/dynamatic/Support/TimingModels.h#L174) is instanciated. This is the top-level class, which contains all timing data of the entire IR. It is initially empty and is then filled (see next section) with instances of the [TimingModel](https://github.com/EPFL-LAP/dynamatic/blob/main/include/dynamatic/Support/TimingModels.h#L103) struct. Each instance of this struct is associated to a single Op of the IR, and contains all the data relative to this Op.

TimingDatabase has a number of internal functions, which can be organised into two categories : the getters, which are called on some Op known to be inside the Database, and which will extract the information from the corresponding TimingModel; and the construction function insertTimingModel, which takes a instanciated TimingModel and adds it into the TimingDatabase.


Within a TimingModel, most fields are directly stored as double,  bitwidth-dependant datafields may have several values for a given operator, and are stored in a custom struct, [BitwidthDepMetric](https://github.com/EPFL-LAP/dynamatic/blob/main/include/dynamatic/Support/TimingModels.h#L46). This struct then in turn has a data field, which contains the map of these differen values, as well as a getCeilMetric function, which serves to extract the value corresponding to a given bitwidth. In order to be callable without prior knowledge of the bitwidth, this function is overloaded; with on version taking the Op as argument, obtaining the assocaited bitwidth, and calling the second overload, which is the version inside BitwidthDepMetric we already mentionned. Of note is that BitwidthDepMetric's data type can be specified freely (and could be maps or complex structs), though the current code only uses on doubles. 

TimingModels can also contain a nested struct PortModel, which is designed to contain the dataDelay, validDelay and readDelay of the Op's input or ouput port. 






## JSON Deserialisation

**The following is a detailed, sequential description of all the relevant steps and functions which build a usable TimingDatabase.**

This process can be summarized as follows :
1 - A TimingDatabase is instanciated, and context passed to it with the MLIRContext getContext function.
2 - readFromJSON is called. this function takes two arguments : the path to the JSON, as (passed as a reference) the instanciated TimindDatabase.  

It's operation can be summarised with the following pseudo-code 

```
function readFromJSON(jsonPath, timingDB):
    open inputFile at jsonPath
    if open fails:
        log error
        return failure

    read all lines from inputFile into a jsonString

    parse jsonString into JSON value
    if parse fails:
        log error
        return failure

    call fromJSON with parsed value and timingDB
    if fromJSON fails:
        return failure

    return success ```

This in turn involves fromJSON. This name is shared by a large number of overloads, in this case it is [fromJSON](https://github.com/EPFL-LAP/dynamatic/blob/main/lib/Support/TimingModels.cpp#L373). Note that the entire component JSON is parsed and passed, not just the relevant operations. 

3- This fromJSON takes the passed JSON content, and reads it key by key, interpreting each key as an Op. it then instantiates for each a TimingModel, and calls a second overload of [fromJSON](https://github.com/EPFL-LAP/dynamatic/blob/main/lib/Support/TimingModels.cpp#L330), which will fill this model's data. Finally, it inserts the model into the general TimingDatabase.

4 - The aforementioned second overload of fromJSON can be described with the following pseudo-code :

```
function fromJSON(value, model, path):
    if value is not an object:
        report error via path
        return false

    // Deserialize scalar fields via deserializeNested
    for each key in [LATENCY, DELAY, DELAY_VALID, DELAY_READY, 
                     DELAY_VR, DELAY_CV, DELAY_CR, DELAY_VC, DELAY_VD]:
        if deserializeNested(key, object, corresponding model field, path) fails:
            return false

    // Deserialize input ports model
    if "inport" exists in object:
        if fromJSON(value["inport"], model.inputModel, path.field("inport")) fails:
            return false
    else:
        report error via path
        return false

    // Deserialize output ports model
    if "outport" exists in object:
        if fromJSON(value["outport"], model.outputModel, path.field("outport")) fails:
            return false
    else:
        report error via path
        return false

    return true
    ```
This fromJSON calls deserializeNested on an desired "out" (the target field of TimingModel) and a fixed list of fields, defined seperately as a list of keys for the JSON; for instance ```DELAY_VALID[] = {"delay", "valid", "1"}```. These match the expected structure of the JSON data, and will be used to extract the appropriate information. The reader wll notice that another fromJSON is called : this third overload of [fromJSON](https://github.com/EPFL-LAP/dynamatic/blob/main/lib/Support/TimingModels.cpp#L314), used on the portModels, simply calls deserializeNested on the elements of the portModel, and will not be discussed independantly as the operating logic is identical to the main case.

5 - [deserializeNested](https://github.com/EPFL-LAP/dynamatic/blob/main/lib/Support/TimingModels.cpp#L256) iterates over the provided list of keys in order to navigate depth-wise the JSON file. Once it reaches the end of the provided list, it then calls fromJSON, taking as arguments both the target out and the built path to the data through the JSON. As we established earlier, since several types exist for fields of TimingModel, this fromJSON could be one of several overloads -currently two are in use, [one](/dynamatic/polygeist/llvm-project/llvm/include/llvm/Support/JSON.h) for doubles, and [one](https://github.com/EPFL-LAP/dynamatic/blob/main/lib/Support/TimingModels.cpp#L283) for BitwidthDepMetric<double>; the first is from JSON.h and simply reads back the value found; the second has the following pseudo-code

```
function fromJSON(value, metric, path):
    object = value.getAsObject()

    for each (key, val) in object:
        bitwidth = bitwidthFromJSON(key)
        dataValue = fromJSON(val)   - this is the fromJSON that operates on doubles

        metric.data[bitwidth] = dataValue

    return true
```

Essentially, it iterates over the found array of values, and writes them one by one into the data field of the BitwidthDepMetric. 


**We have now completed the timing information extraction** : we have initialised a TimingDatabase, available to the functions requiring it; we have populated it with TimingModels for all Ops; and we have filled all the fields of these timingModels." 

## Using the TimingDatabase

The TimingDatabase aims to make available the information stored in components.json. This is done using one of five getters, which are called on an Op and a signalType : getModel, getLatency, getInternalDelay, getPortDelay, getTotalDelay; which respectively pass back a TimingModel, or a double containing the approriate field for the Op. getModel simply finds the name corresponding to the requested Op, and then calls a second getModel which finds the TimingModel corresponding to the name in the database, and returns it; the other getters all first call [getModel]([getModel](https://github.com/EPFL-LAP/dynamatic/blob/main/lib/Support/TimingModels.cpp#L103), and then extract the information from the model's fields.

-**[getTotalDelay](https://github.com/EPFL-LAP/dynamatic/blob/main/lib/Support/TimingModels.cpp#L183)** selects, based off the SignalType, a second getter from {getTotalDataDelay, getTotalValidDelay, getTotalReadyDelay}. getTotalValidDelay and getTotalReadyDelay are internal functions of the model, and directly read and pass back the sum of the relevant fields. However, getTotalDataDelay is bitwidth dependant, and extra logic is required to select the correct value: The field is of type BitwidthDepMetric, therefore we pass desired bitwidth to it's getCeilMetric. This is done for dataDelay of the op, as well as of the nested PortModels, inputModel and outputModel, the results finally being summed.

-**[getPortDelay](https://github.com/EPFL-LAP/dynamatic/blob/main/lib/Support/TimingModels.cpp#L161)** follows the exact same logic, but simply operates on the nested PortModel, and returns as dataDelay only that of the nested PortModel.

-**[getInternalDelay](https://github.com/EPFL-LAP/dynamatic/blob/main/lib/Support/TimingModels.cpp#L143)** likewise shares the same logic, but ignores the nested PortModel structures and only returns the internal data.

-**[getLatency](https://github.com/EPFL-LAP/dynamatic/blob/main/lib/Support/TimingModels.cpp#L114)** targets a bitwidth dependant field, and therefore also relies on getCeilMetric to return the correct value. Two things to note : if the signalType is data, latency is 0 because no information is available for valid and ready signals; if the op is identified as an LSQ, latency is incremented by 3 cycles before being returned because LSQ has roughly 3 extra cycles of latency on loads compared to an MC.




