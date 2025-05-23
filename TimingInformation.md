# Timing Information and its handling in Dynamatic

This document aims to present all essential information relating to the Op timing information, in order to document existing code and ease future implementations which would affect it.

# High-level overview

## What timing information is meant here & its importance

In order to realise a hardware circuit capable of functioning correctly, it is necessary to know to know, for every operator, their **latency** and their **delay**:

  **latency**  is the number of cycles the operator requires to output a result from provided data. It is always an integer.
  
  **delay** is generally the time (indicated in μs) taken for the operator to compute its result from the moment the input data becomes available to the moment the output data is produced. In our case, we distinguish two useful forms of delay : 
    -operator delay, which is delay not comprised between internal registers, which therefore must be accounted for when computing the slack of a path including the operator. Worth 0 for most operators with a latency greater than 0.
    -internal combinational delay, which is the highest delay between two internal registers, which defines the maximum frequency supported by this operator. Not defined for 0 latency operators. 

 

## How this information is stored

## How this information is used

# Implementation details & codebase examples

