# EOC

End-of-Computation (EOC) Controller and Software Simulation for In-Memory Computing

## Table of Contents

- [Introduction](#introduction)
- [EOC](#eoc)
- [Overall Architecture](#overall-architecture)
- [Features](#features)
- [Directory Structure](#directory-structure)
- [Results](#results)

## Introduction
This project includes software simulation and RTL code development to reduce latency in pulse-based in-memory computing (IMC). The software simulation used the CIFAR-10 dataset with the ResNet18 model to test latency reduction through reordering. The controller was synthesized and power measured using the Samsung 28nm LPP process.

For details of the development process, see the [documentation](#https://docs.google.com/presentation/d/1S23ArIcwT69y59zL0pUIKm924a2xaRMqQ_dRSkCHZfM/edit#slide=id.g30d379e08d0_2_19).

## EOC
The IMC stores weights and receives input activations proportional in size to the pulses entering the IMC. When using 4-bit input activations, the maximum pulse count is 15, thus typically requiring a wait of 15 pulse times before operating the ADC. However, analysis of real dataset distributions shows that waiting for 15 pulses is unnecessary as values are often lower. Thus, ADC operations only need to wait for the largest pulse time among the input activations, reducing unnecessary latency. For example, if the IMC processes four input activations at once with values 3, 6, 1, and 2, the ADC operates after just 6 pulses, not 15. For more details, refer to Kwantae's patent.

Despite this optimization, the controller is required to manage variable latency and to load input activations at different timings since all inputs' maximum values determine the latency, potentially lengthening overall latency if values are dispersed. Solutions include double buffering and reordering methods.

## Overall Architecture

The overall architecture is structured as follows:

![image](https://github.com/user-attachments/assets/0fac1615-2886-48c3-81aa-6bdcba903368)



### IMC Macro
The IMC macro consists of a 64x64 cell array, pairing every four rows to form a local cell array (LCA), thus creating 16 LCAs per column, each containing four cells. This configuration allows the IMC to accumulate 16 partial products from different input channels in one operation. To use a 3x3 kernel, the system expands to nine IMC macros for accumulation.

Currently, the IMC macro handles only signal receiving and output in the Verilog code, with actual implementation pending integration with the controller.

### Weight Controller
The weight controller stores weights in the IMC at sizes matching the IMC.

### Input Activation Controller
The input activation memory can read 256 bits at a time from a 64x256 size memory. It employs double buffering and sorting, sorting 64 4-bit data pieces at once, and sending sorted input activations to the IMC in descending order. The sorting index is also sent to determine which weights each LCA should process.

### EOC Block
The EOC block calculates run cycles, adjusting addresses in the input activation controller and generating pulses for the IMC.

### ETC
The IMC pulse operates at five times the speed of the master frequency. This means that during one cycle, five pulses can be generated. Consequently, over three cycles, 15 IMC pulses are produced, and in the final cycle, a signal is generated for the ADC. Therefore, to accumulate 16 pieces of data at once, a maximum of four cycles are required. Below is the timing graph illustrating the master frequency, IMC pulses, and ADC timings.

![image](https://github.com/user-attachments/assets/2ee30928-70da-4a64-b78e-a0ece7350696)



## Features
### Double Buffering & Finding Maximum
Control is necessary since each computation cycle varies, and the cycle for each input is determined by its maximum value. The maximum values are identified from the sorted inputs, calculating the necessary cycles for operations and ADCs. If a maximum is zero, meaning all 16 input activations are zero, no computation or ADC operation is required. Double buffering allows reading data ahead of time, switching buffers instantly after current operations, ensuring continuous processing.

### Reordering
Reordering is facilitated by sorting logic in the input activation, grouping large and small values for more efficient computation, thus mitigating latency even with randomly distributed data.

## Directory Structure

### pt
Primetime for power measurement.

### rtl
Contains RTL code, focusing on internal files like noc, top.

### sim
Enables RTL and SYN simulations.

### sram
Stores necessary SRAM for input activation and weight memories.

### syn
Synthesis can be performed here.

## Results
The waveform of the IMC operation is as follows:
![image](https://github.com/user-attachments/assets/dae0d11c-4883-4cdd-8aa5-ba9f3ab76db0)

The o_imc_pulse comprises 16 pulses, sequenced from largest to smallest values. For instance, observing o_imc_pulse[15] reveals pulse generation in the order of 7, 5, 2, 0 for input activations. During the second phase, only three ADC operations occur, as the final 16 input activations are all zero, leading to their omission. In the third phase, the last and preceding 16 input activations are zero, resulting in only two ADC operations.

Based on the RTL design, latency and power were measured.

### Latency
The controller’s input activation size is 64x256, where 64 denotes the output channels, and 256 represents the direction of 64 4-bit input channels. Since the IMC can process 16 activations simultaneously, operations occur 64 (oCH) x 4 (LCA) times due to the division of 64x256 by 16. In the worst-case scenario, one operation requires 4 cycles. Therefore, performing 256 operations consumes a total of 1024 cycles. However, applying this to the second layer of CIFAR-10 with the ResNet18 model, the operations take only 614 cycles, a 40% reduction in computation cycles.

![image](https://github.com/user-attachments/assets/d050b844-9879-4784-b00f-afb3f1d5140b)


### Power
The power breakdown is as follows:

![image](https://github.com/user-attachments/assets/422f8f82-1ea0-4116-9556-1dcef442f346)

Additions for EOC operation, including reordering, involve eoc, buffers, and sorting components. These additions constitute only 5.3% of the total core consumption, which is 3550uW, at 187.6uW. This indicates minimal overhead due to EOC operations.
