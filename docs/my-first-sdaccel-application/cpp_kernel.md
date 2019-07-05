<table>
 <tr>
   <td align="center"><img src="https://www.xilinx.com/content/dam/xilinx/imgs/press/media-kits/corporate/xilinx-logo.png" width="30%"/><h1>2019.1 SDAccel™ Development Environment Tutorials</h1>
   <a href="https://github.com/Xilinx/SDAccel-Tutorials/branches/all">See other versions</a>
   </td>
 </tr>
 <tr>
 <td align="center"><h1>Create My First SDAccel Program</h1>
 </td>
 </tr>
</table>

# 1. Coding My First C++ Kernel

This lab explains the coding requirements for an SDAccel™ environment kernel. In this lab, you will look at a simple [vector addition kernel](./reference-files/src/vadd.cpp), found in the `reference-files/src` folder.

As mentioned previously, an SDAccel application consists of a software program running on a host CPU and interacting with one or more accelerators running on a Xilinx FPGA. The host program uses API calls to transfer data to and from the FPGA and interact with the accelerators. Data transfers occur by copying data from the host memory to device memory in the FPGA acceleration card. Device memory is subdivided into different memory banks. Accelerators (also referred to as kernels) have memory interfaces that allow them to connect to these memory banks and access data shared with the host program.

In this lab, you will:

1. Learn the interface requirements necessary for a kernel.
2. Understand how function arguments are mapped to the interfaces.
3. Edit the VADD function to turn it into a kernel usable in the SDAccel environment.

## Kernel Interface Requirements

Before you can turn a function into a the kernel, you must first understand the kernel interface requirements. Every C++ function needs to meet certain interface requirements in order to be used as a kernel in the SDAccel development environment. All kernels require the following hardware interfaces:

- A single [AXI4-Lite](https://www.xilinx.com/products/intellectual-property/axi.html#details) slave interface used to access control registers (to start and stop the kernel) and to pass scalar arguments.
- At least one of the following interfaces (the kernel can have both interfaces):
  - [AXI4 Master](https://www.xilinx.com/products/intellectual-property/axi.html#details) interface to communicate with memory.
  - [AXI4-Stream](https://www.xilinx.com/products/intellectual-property/axi.html#details) interface for transferring data between kernels, or between the host and kernel.

Execution of kernels relies on the following mechanics and assumptions:

- Scalar arguments are passed to the kernel through its AXI4-Lite slave interface.
- Pointer arguments are passed to the kernel through its AXI4-Lite slave interface.
- Pointer data resides in global memory (DDR or PLRAM).
- Kernels access pointer data in global memory through one or more AXI4 memory mapped interfaces.
- Kernels are started by the host program through control signals on the kernel's AXI4-Lite interface.

## VADD Example

### C Code Structure

The following code shows the kernel signature of the VADD function in [vadd.cpp](./reference-files/src/vadd.cpp), which uses only pointer and scalar arguments.

```C++
        void vadd(
                const unsigned int *in1, // Read-Only Vector 1 - Pointer arguments
                const unsigned int *in2, // Read-Only Vector 2 - Pointer arguments
                unsigned int *out,       // Output Result      - Pointer arguments
                int size                 // Size in integer    - Scalar arguments
                )
        {
```

### Mapping the Function Arguments

**IMPORTANT:** By default, kernels modeled in C/C++ do not have any inherent assumptions on the physical interfaces that will be used to transport the function arguments. The compiler relies on pragmas embedded in the code to determine which physical interface to generate for each port. For the function to be treated as a valid C/C++ kernel, each function argument should have a valid interface pragma.

#### Pointer Arguments

A pointer argument on the function interface results in two distinct kernel ports. Consequently, each pointer argument requires two interface pragmas.

* In the first port, the kernel will access data in the global memory. This port must be mapped to an AXI Master interface (`m_axi`).
* In the second port, the base address of the data is passed by the host program to the kernel. This port must be mapped to the AXI4-Lite slave interface (`s_axilite`) of the kernel. 

In the VADD kernel, the arguments `in1`, `in2`, and `out` need the following pragmas:

   ```C++
   #pragma HLS INTERFACE m_axi     port=in1 offset=slave bundle=gmem
   #pragma HLS INTERFACE m_axi     port=in2 offset=slave bundle=gmem
   #pragma HLS INTERFACE m_axi     port=out offset=slave bundle=gmem
   #pragma HLS INTERFACE s_axilite port=in1              bundle=control
   #pragma HLS INTERFACE s_axilite port=in2              bundle=control
   #pragma HLS INTERFACE s_axilite port=out              bundle=control
   ```

The `m_axi` interface pragmas are used to characterize the AXI Master ports.

* `port`: Specifies the name of the argument to be mapped to the AXI memory mapped interface.
* `offset=slave`: Indicates that the base address of the pointer is made available through the AXI-Lite slave interface of the kernel.
* `bundle`: Specifies the name of the `m_axi` interface. In this example, the three arguments are mapped to a single AXI interface called `gmem`.

The `s_axilite` interface pragmas are used to characterize the AXI4-Lite port.

#### Scalars

A scalar argument on the function interface results in a port which must to be mapped to the AXI4-Lite interface (`s_axilite`) of the kernel. Note that the function's `return` must be mapped to the AXI4-Lite interface in a similar manner.

In the VADD design, the `size` argument and the `return` value need the following pragmas:

   ```C++
   #pragma HLS INTERFACE s_axilite port=size              bundle=control
   #pragma HLS INTERFACE s_axilite port=return            bundle=control
   ```

Because a kernel can have only one AXI-Lite interface, all `s_axilite` pragmas must use the same `bundle` name. In this example, you are giving the name `control` to the AXI4-Lite interface.

For more information on the HLS INTERFACE pragmas, refer to the _SDx Pragma Reference Guide_ ([UG1253](https://www.xilinx.com/html_docs/xilinx2019_1/sdaccel_doc/nnz1534452175410.html)).

### Using C Linkage

In addition to adding the interface pragmas, you need to add extern "C" { ... } around the functions to be compiled as kernels. The use of extern "C" instructs the compiler/linker to use the C naming and calling conventions.

```C++
extern "C" {
void vadd(
        const unsigned int *in1, // Read-Only Vector 1
        const unsigned int *in2, // Read-Only Vector 2
        unsigned int *out,       // Output Result
        int size                 // Size in integer
        )
 {

#pragma HLS INTERFACE m_axi     port=in1  offset=slave bundle=gmem
#pragma HLS INTERFACE m_axi     port=in2  offset=slave bundle=gmem
#pragma HLS INTERFACE m_axi     port=out  offset=slave bundle=gmem
#pragma HLS INTERFACE s_axilite port=in1               bundle=control
#pragma HLS INTERFACE s_axilite port=in2               bundle=control
#pragma HLS INTERFACE s_axilite port=out               bundle=control
#pragma HLS INTERFACE s_axilite port=size              bundle=control
#pragma HLS INTERFACE s_axilite port=return            bundle=control
```

The `vadd` function can now be compiled into a kernel using the SDAccel toolchain.

## Next Step

The next step in this tutorial is [coding the host program](./host_program.md).

</br>
<hr/>
<p align="center"><b><a href="/docs/sdaccel-getting-started/">Return to Getting Started Pathway</a> — <a href="./README.md">Return to Start of Tutorial</a></b></p>

<p align="center"><sup>Copyright&copy; 2019 Xilinx</sup></p>
