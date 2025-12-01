# OpenCL Programming Guide

## 1. Introduction

**OpenCL (Open Computing Language)** is an open, cross-platform parallel-computing framework maintained by the Khronos Group.  
It provides developers with a unified programming interface that enables applications to run across different hardware platforms — such as CPUs, GPUs, DSPs, and other processors.  
This allows developers to improve **code portability** and **execution performance**.

OpenCL consists of two main parts:
1. **Kernel Language**: A C99-derived language used to write functions (kernels) that execute on OpenCL devices  
2. **Platform API**: Interfaces for defining and controlling the compute platform

![opencl](./static/how_it_works.jpg#pic_center)

As shown in the figure, the OpenCL framework contains two critical API layers:

- **Platform Layer API**  
  Runs on the host (Host) CPU  
  Main functions:  
  - Query and enable the parallel processors or compute devices available in the system  
  - Allow the application to be ported and executed across different systems, supporting a variety of hardware combinations  

- **Runtime API**  
  Core functions:  
  - Compile kernel programs for the selected device  
  - Manage the parallel execution of kernels on the processor  
  - Collect and process the computation results  

## 2. Executing an OpenCL Program

- **Kernel**: The basic execution unit on the device (similar to a C function), supporting two parallel modes:  
  - Data parallelism  
  - Task parallelism  

- **Program Object**: A collection of multiple kernels and functions (analogous to a dynamic library with runtime linking)  

- **Command Queue**: The channel through which the host submits commands to the device, featuring:  
  - In-order / out-of-order execution modes  
  - Support for multiple queues  

The figure below illustrates the flow of executing an OpenCL kernel:

![opencl](./static/executing_programs.jpg#pic_center)

The complete steps for executing an OpenCL program are as follows:

1. Query the available OpenCL platforms and devices  
2. Create a context for the OpenCL devices in one or more platforms  
3. Create and build a program for the devices in the context  
4. Select the kernel(s) to be executed from the program  
5. Create memory objects for the kernel to operate on  
6. Create command queues to issue commands to the OpenCL device  
7. Retrieve the execution results and clean up the environment  

For more details, please refer to:

[OpenCL Guide](https://github.com/KhronosGroup/OpenCL-Guide)

## Key APIs

### OpenCL Platform

Selecting an OpenCL platform is the first step in OpenCL. The API `clGetPlatformIDs()` is used to discover the set of available OpenCL platforms on a given system.
`cl_int clGetPlatformIDs(cl_uint num_entries, cl_platform_id *platforms, cl_uint *num_platforms)`

- `num_entries`: the number of OpenCL platforms to return.  
  When set to `0` and `platforms` is `NULL`, it queries the number of available platforms.

- `platforms`: pointer to the list of returned platform IDs.

- `num_platforms`: returns the actual number of OpenCL platforms available.

This API is typically called twice: first to query the number of platforms, then to retrieve their IDs, as shown below:

```c
cl_int err = 0; // Error Codes
cl_uint num_platform = 0; // Number of Platforms
cl_platform_id *platform = NULL; // Platform ID Pointer
err = clGetPlatformIDs(0, NULL, &num_platform); // First parameter: number of platforms to query, Second parameter: array of platform IDs, Third parameter: returned platform count
if (err!= CL_SUCCESS) { // Check for errors
    fprintf(stderr, "Failed to create context: %d\n", err); // Output error message
    exit(-1); // Exit program
}
platform = (cl_platform_id*)malloc(sizeof(cl_platform_id) * num_platform); // Allocate memory to store platform IDs
err = clGetPlatformIds(num_platform, platform, NULL); // Retrieve platform IDs
```

### OpenCL Device

Once the platform is determined, the next step is to query the available devices on that platform:

```c
/**
 * Function for retrieving device IDs
 * @return `cl_int` error code
 */
cl_int clGetDeviceIDs(
    cl_platform_id platform,  //Platform ID
    cl_device_type device_type,  //Device type
    cl_uint num_entries,  //Number of device IDs to retrieve
    cl_device_id *devices,  //Array for storing device IDs
    cl_uint *num_devices  //Actual number of device IDs retrieved
);
// Usage:
cl_int err = 0;  // Used to store the error code
cl_uint num_devices = 0;  // Used to store the number of devices
cl_device_id *devices = NULL;  // Pointer to store device IDs
err = clGetDeviceIDs(platform, CL_DEVICE_TYPE_GPU, 0, NULL, &num_devices);  // Retrieve the number of GPU devices: `platform` is the platform, `CL_DEVICE_TYPE_GPU` indicates GPU devices, `0` means no specific device is requested, `NULL` means no device-ID list is returned, and `&num_devices` stores the device count.
if (err!= CL_SUCCESS)  // Check for errors
    exit(-1);  // If an error occurs, exit the program
devices = (cl_device_id*)malloc(sizeof(cl_device_id) * num_devices);  // Allocate memory for device IDs
err = clGetDeviceIDs(platform, CL_DEVICE_TYPE_GPU, num_devices, devices, NULL);  // Retrieve the list of device IDs
```

The `cl_device_type` parameter is described as follows:

- `CL_DEVICE_TYPE_CPU`: Uses the CPU as an OpenCL device  
- `CL_DEVICE_TYPE_GPU`: GPU device  
- `CL_DEVICE_TYPE_ACCELERATOR`: Accelerator devices, e.g., FPGA devices (OpenCL devices of the accelerator type)  
- `CL_DEVICE_TYPE_DEFAULT`: The default OpenCL device associated with the platform  
- `CL_DEVICE_TYPE_ALL`: All OpenCL devices supported by the platform

### OpenCL Context

In OpenCL, a context coordinates and manages memory to ensure kernels execute correctly. A context object is created with `clCreateContext()`.

```c
// Create an OpenCL context
cl_context clCreateContext(
    const cl_context_properties *properties,  // Context property list
    cl_uint num_devices,  // Number of devices
    const cl_device_id *devices,  // Array of device IDs
    void (CL_CALL_BACK *pfn_notify)(const char *errinfo, const void *private_info, size_t cb, void *user_data),  // A user-defined callback function, paired with `user_data`, that reports any errors occurring during the context’s lifetime.
    void *user_data,  // User-supplied data that will be passed to the error-notification callback function.
    cl_int *errcode_ret  // Pointer used to return the error code
);
```

OpenCL provides another API to create a context:  
`clCreateContextFromType()` can create a context using all devices of a specified type (`CPU`, `GPU`, or `ALL`).

### OpenCL Command Queue

Operations on a context’s **program objects**, **memory objects**, and **kernel objects** must be performed through a command queue.  
Commands are messages sent from the **host** to **devices**, instructing the devices to carry out operations.  
Each command queue can manage only one device.

OpenCL’s `clCreateCommandQueueWithProperties()` is used to create a command queue and associate it with a device, as shown below:

```c
// Create a command queue with specific properties
cl_command_queue clCreateCommandQueueWithProperties(
    cl_context context, // Context object used to associate the command queue with a device
    cl_device_id device, // Device to be associated
    cl_command_queue_properties properties, // Properties for the command queue, such as enabling out-of-order execution or profiling; defaults to in-order execution.
    cl_int *errcode_ret // Pointer used to return the error code
);
```

### OpenCL Program Object and Kernel Object

**Program objects** and **kernel objects** are the most important parts of OpenCL.  
A program object is a container for kernels; one program object can hold multiple kernel objects, and kernel objects are created and managed by the program object.

An OpenCL program object typically contains:
- One or more kernel functions written in OpenCL C;  
- Auxiliary functions that those kernels call;  
- Constant data.

For example, in an algebraic-computation scenario, a single program object might include all three of the following kernels:
- A kernel for vector addition  
- A kernel for matrix multiplication  
- A kernel for matrix transposition  

The steps to create kernels from source code are as follows:

1. **Prepare the source code:**  
   Store the OpenCL C source in a character array.  
   If the source resides in a file on disk, read the file contents into a character array in memory first.

2. **Create the program object:**  
   Call `clCreateProgramWithSource()` to create a program object of type `cl_program` from the source code.

3. **Build the program object:**  
   The created program object must be compiled before its kernels can run on one or more OpenCL devices.  
   Invoke `clBuildProgram()` to compile the kernels; if any compilation issues occur, this API will output diagnostic messages.

4. **Create kernel objects:**  
   Finally, create kernel objects `cl_kernel`.  
   Call `clCreateKernel()`, specifying the corresponding program object and the kernel function name, to instantiate the kernel object.

A kernel object is essentially a function. It has parameters and (optionally) a return value, which are passed in and out via memory objects. This function executes on an OpenCL device.

**Kernel source code example for vector addition:**

```c
// Perform an element-wise addition of A and B and store in C.
// N work-items will be created to execute this kernel.
__kernel
void vecadd(__global int *C, __global int *A, __global int *B){
  int tid = get_global_id(0); // OpenCL Intrinsic Functions
  c[tid] = A[tid] + B[tid];
}
```

**Create a program object:**

```c
cl_program clCreateProgramWithSource(
    cl_context context,          // Context object
    cl_uint count,               // Number of source-code strings
    const char **strings,        // Array of source-code strings
    const size_t *lengths,       // Array of the length of each source-code string
    cl_int *errcode_ret          // Return error code
    )
```

**Build the program object:**

```c
@return  Compiled program object */
cl_int clBuildProgram(
    cl_program program,               // Program object to be built
    cl_uint num_devices,              // Number of devices
    const cl_device_id *device_list,  // List of devices
    const char *options,              // Build options
    void (*pfn_notify)(cl_program,    // Callback function
                       void *user_data),
    void *user_data                   // User data
)
```

**Create a kernel object:**

```c
@return  The created kernel object */
cl_kernel
clCreateKernel(
    cl_program program,      // Program object from which to create the kernel
    const char *kernel_name, // Kernel name (i.e., the kernel function name)
    cl_int *errcode_ret      // Pointer for returning the error code
)
```

### OpenCL Memory Objects

OpenCL kernels usually need to process input and output data such as arrays or multi-dimensional matrices.  
Before execution, you must ensure the input data is accessible on the device. To transfer data to the device:

1. Allocate sufficient space on the device;  
2. Wrap that space in a memory object so the kernel can access it.

OpenCL defines three memory types: **buffer**, **image**, and **pipe**.

**Buffer Creation**  
A buffer stores contiguous data in memory and can be accessed on the device via a pointer.  
`clCreateBuffer()` allocates memory for this type and returns a memory object.

```c
cl_mem clCreateBuffer(
    cl_context context,   // Context object that associates the device and command queue
    cl_mem_flags flags,   // Memory-object flags, e.g. CL_MEM_READ_WRITE for read/write
    size_t size,          // Size of the memory object in bytes
    void *host_ptr,       // Host pointer; if NULL, memory is allocated on the device
    cl_int *errcode_ret   // Pointer to receive an error code; may be NULL
)
```

**Kernel Parameter Setup**

Unlike a regular C function, an OpenCL kernel’s arguments cannot be placed directly in its parameter list.  
At run time, arguments must be passed via enqueued commands.

> **Note:** Kernel syntax is based on C, and arguments have **persistence**.  
> This means that if only the contents of an argument are modified (rather than rebinding a new memory object), you do **not** need to call the setter function again.

In OpenCL, `clSetKernelArg()` is used to set a kernel’s arguments.

```c
cl_int clSetKernelArg(
    cl_kernel kernel,      // Kernel object whose argument is to be set
    cl_uint arg_index,     // Index of the kernel argument, starting from 0
    size_t arg_size,       // Size of the argument in bytes
    const void *arg_value  // Pointer to the argument value
)
```

### OpenCL Kernel Execution & Error Handling
 
Calling `clEnqueueNDRangeKernel()` places a kernel-execution command into the command queue.
The command queue specifies the target device, while the kernel object determines the actual code to run.

During kernel execution, work-items are created using four key parameters:

- `work_dim` – dimensionality of the NDRange (1-D, 2-D, or 3-D).  
- `global_work_size` – number of work-items in each dimension of the NDRange.  
- `local_work_size` – size of each work-group in each dimension.  
- `global_work_offset` – initial offset applied to the global work-item IDs (whether counting starts from 0).

```c
// Function: enqueue the kernel for execution
cl_int clEnqueueNDRangeKernel(
    cl_command_queue command_queue,      // Command queue in which the kernel is executed
    cl_kernel        kernel,             // Kernel to execute
    cl_uint          work_dim,           // Number of dimensions in the NDRange
    const size_t    *global_work_offset, // Global work-item ID offset (can be NULL)
    const size_t    *global_work_size,   // Global work-item size per dimension
    const size_t    *local_work_size,    // Work-group size per dimension (can be NULL)
    cl_uint          num_events_in_wait_list, // Number of events to wait for
    const cl_event  *event_wait_list,    // Array of events to wait for
    cl_event        *event               // Returns an event object for this kernel run
);
```

After execution is complete, retrieve the result:

```c
cl_int clEnqueueReadBuffer(
    cl_command_queue command_queue,  // Command queue
    cl_mem buffer,                   // Buffer object to read from
    cl_bool blocking_read,           // Blocking or non-blocking read flag
    size_t offset,                   // Offset in the buffer, typically set to 0
    size_t cb,                       // Size of the memory to read
    void* ptr,                       // Pointer on the host to receive the data
    cl_uint num_events_in_wait_list, // Number of events in the wait list
    const cl_event *event_wait_list, // List of events to wait for
    cl_event *event                  // Pointer to retrieve the event
)
 
```

Finally, use `clRelease` to release all resources:

```c
clReleaseKernel(kernel);             // Release kernel object
clReleaseProgram(program);           // Release program object
clReleaseCommandQueue(cmdQueue);     // Release command queue object
clReleaseMemObject(bufA);            // Release memory object bufA
clReleaseMemObject(bufB);            // Release memory object bufB
clReleaseMemObject(bufC);            // Release memory object bufC
clReleaseContext(context);           // Release context object
```

For more detailed API usage, refer to:

1. [OpenCL 3.0 Reference Guide](https://www.khronos.org/files/opencl30-reference-guide.pdf)

2. [OpenCL Reference Pages](https://registry.khronos.org/OpenCL/sdk/3.0/docs/man/html/)

3. [The OpenCL™ Specification](https://registry.khronos.org/OpenCL/specs/3.0-unified/html/OpenCL_API.html)

## OpenCL Demo

### Introduction

In the **Buildroot** system, the demo source code is located at:

```
xxx/buildroot-sdk/package-src/k1x-gpu-test/openGLDemo
```

In the **Bianbu** system, you can install `k1x-gpu-test` using the following command to get the related demo:

```bash
sudo apt install k1x-gpu-test
```

The demo directory structure is as follows:

```c
.
|-- add_demo.c //Vector Addition Demo
|-- CMakeLists.txt //Used for compilation and building
`-- README.md

0 directories, 3 files
```

### Compile & Run

```bash
sudo apt install opencl-headers ocl-icd-opencl-dev #Install dependencies
cd k1x-gpu-test/openCLDemo
cmake .
make -j
```

After compilation, the `gpu-addDemo` file will be generated in the current directory. Simply execute it to run the demo:

```bash
./gpu-addDemo
```

If installation is needed, you can run the `make install` command in the current directory (the source directory).  
This will automatically install the executable to the `./usr/local/bin/` directory.  
After installation, you can run `gpu-addDemo` directly in the terminal.  
The correct output should be as follows:

```shell
65024.000000 65028.000000 65032.000000 65036.000000 65040.000000 65044.000000 65048.000000 65052.000000 65056.000000 65060.000000 65064.000000 65068.000000 65072.000000 65076.000000 65080.000000 65084.000000 65088.000000 65092.000000 65096.000000 65100.000000 65104.000000 65108.000000 65112.000000 65116.000000 65120.000000 65124.000000 65128.000000 65132.000000 65136.000000 65140.000000 65144.000000 65148.000000 65152.000000 65156.000000 65160.000000 65164.000000 65168.000000 65172.000000 65176.000000 65180.000000 65184.000000 65188.000000 65192.000000 65196.000000 65200.000000 65204.000000 65208.000000 65212.000000 65216.000000 65220.000000 65224.000000 65228.000000 65232.000000 65236.000000 65240.000000 65244.000000 65248.000000 65252.000000 65256.000000 65260.000000 65264.000000 65268.000000 65272.000000 65276.000000 65280.000000 65284.000000 65288.000000 65292.000000 65296.000000 65300.000000 65304.000000 65308.000000 65312.000000 65316.000000 65320.000000 65324.000000 65328.000000 65332.000000 65336.000000 65340.000000 65344.000000 65348.000000 65352.000000 65356.000000 65360.000000 65364.000000 65368.000000 65372.000000 65376.000000 65380.000000 65384.000000 65388.000000 65392.000000 65396.000000 65400.000000 65404.000000 65408.000000 65412.000000 65416.000000 65420.000000 65424.000000 65428.000000 65432.000000 65436.000000 65440.000000 65444.000000 65448.000000 65452.000000 65456.000000 65460.000000 65464.000000 65468.000000 65472.000000 65476.000000 65480.000000 65484.000000 65488.000000 65492.000000 65496.000000 65500.000000 65504.000000 65508.000000 65512.000000 65516.000000 65520.000000 65524.000000 65528.000000 65532.000000
Start time: 1732783455 sec, 480682 usec
End time: 1732783455 sec, 699529 usec
Calculate time for 102400000 addition operations: 218847 us
```

### Add Demo

If you add a file named `testDemo.c` and want to compile it into an executable named `testDemo`, you can modify `CMakeLists.txt` as shown in the following example:

```CMake
# Define OpenCL version as 300
add_definitions(-DCL_TARGET_OPENCL_VERSION=300)

# Add linked libraries
set(LINK_LIBRARIES OpenCL)  # Set the link libraries variable, specifying the library name

add_executable(gpu-addDemo ${CMAKE_CURRENT_SOURCE_DIR}/add_demo.c)
add_executable(testDemo ${CMAKE_CURRENT_SOURCE_DIR}/testDemo.c)  # Create a new executable

# Link libraries, link the specified libraries with the executables
target_link_libraries(gpu-addDemo ${LINK_LIBRARIES})
target_link_libraries(testDemo ${LINK_LIBRARIES})

# Install, install executables to the specified directory
install(TARGETS 
gpu-addDemo
testDemo  # Install the new executable
DESTINATION "${CMAKE_INSTALL_PREFIX}/bin")
```

Then run `make -j` again.

## Others

### Using `printf` in Kernel

Due to hardware limitations of the GPU, the maximum size of the `printf` buffer in OpenCL is about **360,000 Bytes**.  
When using the `printf` function in an OpenCL Kernel, you must escape special characters (e.g., `\n`) in the string, as shown below:

```c
const char *programSource =
    "__kernel                                         \n"
    "void vec_add(__global float *A,                  \n"
    "             __global float *B,                  \n"
    "             __global float *C)                  \n"
    "{                                                \n"
    "  // Get the work-item's unique ID               \n"
    "  int idx = get_global_id(0);                    \n"
    "  printf(\"test\");                              \n"
    "}                                                \n";

```

If the source code is imported via an external `xxx.cl` file, there is no need to add escape characters.
