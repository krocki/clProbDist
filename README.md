# clProbDist Library

A library for probability distributions in OpenCL.


## Overview

The clProbDist library provides facilities for evaluating, on OpenCL devices and on hosts:

- probability density functions;
- probability mass functions;
- cumulative distribution functions;
- inverse cumulative distribution functions; and
- reliability functions.

Five distributions are currently supported:

- normal;
- lognormal;
- exponential;
- gamma; and
- Poisson.

In its current state, clProbDist defines a framework for working with
probability distributions in OpenCL, and to show some working examples.  Dozens
of **additional distributions need to be contributed** to make clProbdist a
mature library.


## Documentation

The
[clProbDist documentation](http://umontreal-simul.github.io/clProbDist/htmldocs/index.html)
includes:

- [a reference for the implemented distributions](http://umontreal-simul.github.io/clProbDist/htmldocs/index.html#distributions);
- [basic usage examples](http://umontreal-simul.github.io/clProbDist/htmldocs/index.html#basic_usage) including the [generation of nonuniform variates](http://umontreal-simul.github.io/clProbDist/htmldocs/index.html#example_poissongen) with the help of the [clRNG library](https://github.com/clMathLibraries/clRNG); and
- [the API reference](http://umontreal-simul.github.io/clProbDist/htmldocs/clProbDist__template_8h.html).


## Examples

Examples can be found in `src/client`.
The compiled client program examples can be found under the `bin` subdirectory
of the installation package (`$CLPROBDIST_ROOT/bin` under Linux).


## Simple example

The simple example below shows how to use clProbDist to compute the inverse CDF
of the exponential distribution using device side headers (`.clh`) in your
OpenCL kernel.
Note that the example expects an OpenCL GPU device, that supports
double-precision floating point computations, to be available.
This example uses the API that uses *distribution objects*; another API that uses distribution parameters instead is also available.

```c
#include <stdlib.h>
#include <string.h>
#include <stdio.h>

#include <clProbDist/clProbDist.h>
#include <clProbDist/exponential.h>


int main( void )
{
    cl_int err;
    cl_platform_id platform = 0;
    cl_device_id device = 0;
    cl_context_properties props[3] = { CL_CONTEXT_PLATFORM, 0, 0 };
    cl_context ctx = 0;
    cl_command_queue queue = 0;
    cl_program program = 0;
    cl_kernel kernel = 0;
    cl_event event = 0;
    cl_mem bufIn, bufOut;
    double *out;
    const char *includes;
    char buildLog[4096];
    size_t numWorkItems = 64;
    size_t i;
    clprobdistExponential *dist = 0;
    size_t distBufferSize = 0;
    size_t kernelLines = 0;

    /* Sample kernel that calls clProbDist device-side interfaces to compute the
       inverse CDF of the exponential distribution at different quantiles */
    const char *kernelSrc[] = {
        "#include <clProbDist/exponential.clh>                             \n",
        "                                                                  \n",
        "__kernel void example(__global const clprobdistExponential *dist, \n",
        "                      __global double *out)                       \n",
        "{                                                                 \n",
        "    int gid = get_global_id(0);                                   \n",
        "    int gsize = get_global_size(0);                               \n",
        "    double quantile = (gid + 0.5) / gsize;                        \n",
        "    out[gid] = clprobdistExponentialInverseCDFWithObject(         \n",
        "                                      dist, quantile, (void *)0); \n",
        "}                                                                 \n",
    };

    /* Setup OpenCL environment. */
    err = clGetPlatformIDs( 1, &platform, NULL );
    err = clGetDeviceIDs( platform, CL_DEVICE_TYPE_GPU, 1, &device, NULL ); // FIXME
    err = clGetDeviceIDs( platform, CL_DEVICE_TYPE_CPU, 1, &device, NULL );

    props[1] = (cl_context_properties)platform;
    ctx = clCreateContext( props, 1, &device, NULL, NULL, &err );
    #ifdef CL_VERSION_2_0
        queue = clCreateCommandQueueWithProperties( ctx, device, (cl_queue_properties[]){0}, &err );
    #else
        queue = clCreateCommandQueue( ctx, device, 0, &err );
    #endif


    /* Make sure CLPROBDIST_ROOT is specified to get library path */
    includes = clprobdistGetLibraryDeviceIncludes(&err);
    if(err != CL_SUCCESS) printf("\n%s\nSpecify environment variable CLPROBDIST_ROOT as described\n", clprobdistGetErrorString());

    /* Create sample kernel */
    kernelLines = sizeof(kernelSrc) / sizeof(kernelSrc[0]);
    program = clCreateProgramWithSource(ctx, kernelLines, kernelSrc, NULL, &err);
    err = clBuildProgram(program, 1, &device, includes, NULL, NULL);
    if(err != CL_SUCCESS)
    {
        printf("\nclBuildProgram has failed\n");
        clGetProgramBuildInfo(program, device, CL_PROGRAM_BUILD_LOG, 4096, buildLog, NULL);
        printf("%s", buildLog);
        exit(1);
    }
    kernel = clCreateKernel(program, "example", &err);

    /* Create an exponential distribution with unit mean */
    dist = clprobdistExponentialCreate(1.0, &distBufferSize, (clprobdistStatus *)&err);

    /* Create buffers for the kernel */
    bufIn = clCreateBuffer(ctx, CL_MEM_READ_ONLY | CL_MEM_COPY_HOST_PTR, distBufferSize, dist, &err);
    bufOut = clCreateBuffer(ctx, CL_MEM_WRITE_ONLY | CL_MEM_HOST_READ_ONLY, numWorkItems * sizeof(cl_double), NULL, &err);

    /* Setup the kernel */
    err = clSetKernelArg(kernel, 0, sizeof(bufIn),  &bufIn);
    err = clSetKernelArg(kernel, 1, sizeof(bufOut), &bufOut);

    /* Execute the kernel and read back results */
    err = clEnqueueNDRangeKernel(queue, kernel, 1, NULL, &numWorkItems, NULL, 0, NULL, &event);
    err = clWaitForEvents(1, &event);
    out = (double *)malloc(numWorkItems * sizeof(out[0]));
    err = clEnqueueReadBuffer(queue, bufOut, CL_TRUE, 0, numWorkItems * sizeof(out[0]), out, 0, NULL, NULL);

    /* Display results and check that the original quantile is recovered by evaluating the CDF */
    for (i = 0; i < numWorkItems; i++)
        printf("quantile %.3f :   %9.3e   ->   %.3f\n", (i + 0.5) / numWorkItems, out[i], clprobdistExponentialCDFWithObject(dist, out[i], &err));

    /* Release allocated resources */
    clReleaseEvent(event);
    free(out);
    clReleaseMemObject(bufIn);
    clReleaseMemObject(bufOut);

    clReleaseKernel(kernel);
    clReleaseProgram(program);

    clReleaseCommandQueue(queue);
    clReleaseContext(ctx);

    return 0;
}
```


## Acknowledgments

clProbDist was developed by Nabil Kemerchou, David Munger and Pierre L'Ecuyer at
Université de Montréal, in collaboration with Advanced Micro Devices, Inc.
