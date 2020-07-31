# U200_to_F1

This lab demonstrates how to port an Vitis application developed for U200 card and steps on how to migrate this application to an F1 instance. 

In this lab, you will start with existing project that has been compiled for U200, review the compilation/linking script first. Next, you will review the required modifications for porting to F1 instance. Finally, you will generate the AFI that can be loaded on the F1 instane.


## Overview of the Application 

Vitis core development application consists of a software program running on a host CPU and interacting with one or more accelerators running on a Xilinx FPGA. In this lab, the accelerator has functionality of a simple vector addition. The project comprises of multiple files under src directory.
- `vadd.cpp` contains the software implementation of the kernel which performs simple addition of 2 input values. 
- `host.cpp` contains the main function running on host CPU. The host application is written in either C++ using OpenCL APIs calls to interact with the FPGA accelerator.
There are separate directories for building the kernel for `u200 card` and `F1 instance`, u200 and f1 respectively. 

## Build U200 Kernel
Change to directory `u200` for building kernel for F1 instance. 
The software program is written in C/C++ and uses OpenCL™ API calls to communicate and control the accelerated kernels. Use the following command to compile the program using GCC/g++ compiler

```g++ -D__USE_XOPEN2K8 -D__USE_XOPEN2K8 -I/$(XRT_PATH)/opt/xilinx/xrt/include/ -I./src -O3 -Wall -fmessage-length=0 -std=c++11 ../src/host.cpp -L/$(XRT_PATH)/opt/xilinx/xrt/lib/ -lxilinxopencl -lpthread -lrt -o build_u200/host```

The following `v++ compile` command creates .xo file using stanard v++ compile options to add profiling and location of directories for saving reports. Some of the key options to note here. 
- `-plaform` sets the targetted plaform. For U200 card, its set to xilinx_u200_xdma_201830_2

```v++ -c -g -t hw -R 1 -k vadd --platform xilinx_u200_xdma_201830_2 --profile_kernel data:all:all:all --profile_kernel stall:all:all:all --save-temps --temp_dir ./temp_dir --report_dir ./report_dir --log_dir ./log_dir -I../src ../src/vadd.cpp -o ./vadd.hw_emu.xo```

The following `v++ linker` command creates xclbin which can be loaded on the FPGA. 

```v++ -l -g -t hw -R 1 --platform xilinx_u200_xdma_201830_2 --profile_kernel data:all:all:all --profile_kernel stall:all:all:all --temp_dir ./temp_dir --report_dir ./report_dir --log_dir ./log_dir  --config ./connectivity.cfg -I../src vadd.hw_emu.xo -o add.hw_emu.xclbin```

-   Kernel arguments are connected to the same bank DDR1 and `-sp` option is used to enable this linking. This option is added in connectivity section of `connectivity.cfg` file as shown below.
    ```[connectivity]
    sp=vadd_1.in1:DDR[1]
    sp=vadd_1.in2:DDR[1]
    sp=vadd_1.out:DDR[1]
    ```
Above commands are encapsulated in Makefile targets. You can execute ```make build``` that will build the host, kernel and finally create the xclbin for u200. 
    
## Build F1 kernel - Compilation and Linking script changes 
Change to directory `f1` for building kernel for F1 instance. For this application to run on F1 instance, you will need to make the following changes only.
1. Modify the target platform to AWS platform by updatig v++ compile and linking scripts to the following 
    `-platform $(AWS_PLATFORM)` which is set to xilinx_aws-vu9p-f1_shell-v04261818_201920_2

2. Connect the kernel arguments connection to DDR[0] by updating `connectivity.cfg` as shown below
    ```[connectivity]
    sp=vadd_1.in1:DDR[0]
    sp=vadd_1.in2:DDR[0]
    sp=vadd_1.out:DDR[0]
    ```

You can execute ```make build``` that will build the host, kernel and finally create the xclbin. Next, you will need to create `awsxclbin` that can be loaded on F1 instance.

3. Run the following script to generate `awsxclbin`

``` $VITIS_TOOLS/create_vitis_afi.sh -xclbin=./vadd.xclbin -o=./vadd -s3_bucket=<bucket-name> -s3_dcp_key=f1-dcp-folder -s3_logs_key=f1-logs ```

-   THe previous step will create *_afi_id.txt file. Open this file and record the `AFI Id`. Check the AFI creation status using AFI ID as shown below
    `aws ec2 describe-fpga-images --fpga-image-ids <AFI ID>`
-   If the state is shown as 'available', it indicates AFI creation is completed.  
    "Code": "available"

## Run the Application on F1

1. Execute the following command to source the Vitis runtime environment 
```source $AWS_FPGA_REPO_DIR/vitis_runtime_setup.sh```
2. Execute the host application with the .awsxclbin FPGA binary
``` ./vadd vadd.awsxclbin ``` 
3. Above command displays that the Hardware run is Passing as shown below.


## Conclusion 


