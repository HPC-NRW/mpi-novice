---
title: "Basic MPI Concepts"
teaching: 30
exercises: 0
---

:::::::::::::::::::::::::::::::::::::: questions

- What is the basic lifecycle of an MPI application?
- How are MPI processes identified and organized?
- How are MPI programs compiled and launched on HPC systems?
- What are the main MPI communication paradigms?

::::::::::::::::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::: objectives

- Describe the initialization and finalization phases of an MPI program in C and Fortran.
- Explain the roles of communicators and ranks in MPI.
- Recognize common MPI compiler wrappers and launch mechanisms on HPC systems.
- Distinguish between blocking/non‑blocking and synchronous/asynchronous MPI operations.
- List the three main MPI communication paradigms.

::::::::::::::::::::::::::::::::::::::::::::::::

## Introduction

In this episode, we will discuss some core concepts of MPI.

MPI is a *library interface*, not a language extension. This has several consequences:

- You have to initialize MPI explicitly.
- MPI programs follow a characteristic lifecycle.
- The MPI library can be implemented for many languages and compilers without changing the languages themselves.

We will first look at this lifecycle using a skeleton C program, then briefly contrast this with the Fortran case, and finally discuss some general MPI concepts.


## The Classic MPI Lifecycle in C

### Pre-initialization: before `MPI_Init`

To make MPI functions, types, and constants known to the C compiler, you have to include the MPI header:

```c
#include <mpi.h>
```

Include `mpi.h` in every source file that uses MPI functions, types, or constants.

In C, the first user function that gets executed is `main`. At this point, MPI is **not** yet initialized. There are strict restrictions on which MPI functions may be called before MPI is initialized. These restrictions depend on the MPI version your implementation supports; later MPI standards relaxed some of them.

For the *classic* initialization mode discussed here (no user threads, i.e. a single-threaded MPI program), we assume the strictest rules:

- Before `MPI_Init` completes, you **must not** call any MPI routine except:
  - routines that test whether MPI has been initialized, and
  - routines that actually initialize the library (such as `MPI_Init`).

In this phase, you *may* use MPI types and constants to define and initialize variables, but you must not call MPI communication routines.

Because of these restrictions, `MPI_Init` is usually called very early in the application lifecycle to minimize the phase in which MPI cannot be used.

### Initialization: `MPI_Init`

In C, you initialize MPI with a call such as:

```c
int main(int argc, char *argv[]) {
    MPI_Init(&argc, &argv);
    /* ... */
}
```

You pass the *addresses* of `argc` and `argv` to `MPI_Init` because the MPI library is allowed to modify the command-line argument list. For example, the launcher may have added its own command-line options which MPI will remove before your application processes the arguments.

During `MPI_Init`:

- Several MPI objects with global scope are created.
- Typically, all MPI processes synchronize globally once.
- The communicator `MPI_COMM_WORLD` is initialized; we will discuss communicators in more detail later.

After `MPI_Init` returns, MPI is fully initialized and the *parallel phase* of the application begins.

### Parallel phase

After initialization:

- All MPI processes are active.
- The program typically spends most of its runtime in this phase.
- All MPI communication and computation intended to be parallel takes place here.

### Finalization: `MPI_Finalize`

When the application is about to terminate, the MPI library must be *finalized* explicitly:

```c
MPI_Finalize();
```

Even though applications usually do not do much after finalization other than local cleanup, calling `MPI_Finalize` is important for at least two reasons:

1. **Resource cleanup**  
   MPI may have allocated special memory or other resources that will not be properly released if the process simply exits without finalizing. This can affect the execution environment on the node.

2. **Tools and analysis**  
   Performance analysis and correctness tools commonly intercept `MPI_Finalize` to write out their results. Failing to call `MPI_Finalize` can therefore result in missing or incomplete performance and correctness reports.

After finalization:

- No further MPI function calls are allowed.
- The *post-finalization* phase is usually very short and should be used only for local (non-MPI) cleanup.

An important subtlety: according to the MPI standard, only **rank 0** in the global scope of MPI processes (in `MPI_COMM_WORLD`) is *guaranteed* to return from `MPI_Finalize` to the user program. For other ranks, the behavior is implementation dependent.

In practice, most modern MPI implementations return from `MPI_Finalize` on all ranks, but you should **not** rely on this. Do not write code whose correctness depends on implementation-specific behavior.


## MPI Lifecycle in Fortran

On the Fortran side, the overall situation is similar, but there are important differences.

### Modules instead of headers

In Fortran you do not include a header. Instead, you use a pre-compiled MPI *module*. The actual module names depend on your MPI implementation. Common modules are:

- `use mpi_f08` – the newest Fortran 2008 bindings (recommended, usually named `mpif08` or `mpi_f08`).
- `use mpi` – the Fortran 90 bindings, which are still widely available.

### Initialization and error codes

The pre-initialization phase is similar to C: MPI has not yet been initialized and only a very restricted set of MPI routines may be called.

Initialization differs in two aspects:

1. **Subroutine calls with explicit error codes**  
   In Fortran, the MPI bindings are defined as procedures (subroutines) that must be called with the `call` keyword. The error code is returned in an additional integer argument, for example:

   ```fortran
   use mpi_f08
   implicit none

   integer :: ierr

   call MPI_Init(ierr)
   ```

2. **No `argc` / `argv` arguments**  
   Fortran does not have built-in `argc` and `argv` arguments in the same sense as C. Consequently, the Fortran binding of `MPI_Init` does not include command-line arguments.

The remaining parts of the lifecycle—parallel phase, finalization, and post-finalization—are analogous to the C case.


## Summary: `MPI_Init` and `MPI_Finalize`

Let us summarize the two central calls:

- `MPI_Init`:
  - Initializes the MPI library.
  - Makes the calling process part of the communicator `MPI_COMM_WORLD`.
  - Must be called **exactly once** in an MPI application.
  - In C, you may pass `NULL` as the arguments if you do not need command-line processing, similar to defining `main` without arguments.
  - Returns an error code as the function return value in C, and in an additional integer argument in Fortran.

- `MPI_Finalize`:
  - Must also be called **exactly once** during the lifetime of an MPI application.
  - Cleans up MPI internal resources and allows tools to complete their analysis.
  - After it returns, no further MPI calls are allowed.


## Communicators and Ranks

In the first episode of this tutorial, we introduced the SPMD (Single Program, Multiple Data) model and motivated the need for a unique identification of processes to guide data decomposition and control flow.

We have already encountered the communicator `MPI_COMM_WORLD`, which is initialized during `MPI_Init`.

### Communicators

A **communicator** is the core communication context in MPI. It provides:

- A *group* of processes that can communicate with each other.
- Additional internal information, such as topology information (which we will discuss later).
- A *logical context* for communication operations.

The communicator `MPI_COMM_WORLD` contains all processes that were initially launched for the MPI application.

### Determining the number of processes

To determine how many processes are part of a communicator, use:

- In C:

  ```c
  int size;
  MPI_Comm_size(MPI_COMM_WORLD, &size);
  ```

- In Fortran (Fortran 90 style):

  ```fortran
  integer :: size, ierr
  call MPI_Comm_size(MPI_COMM_WORLD, size, ierr)
  ```

This enables **dynamic** handling of the process count. You do not need to compile the number of processes into your application; instead, you can query the actual number of processes provided at runtime.

### Determining the rank (ID) of a process

To determine the rank (the unique ID) of a process within a communicator, use:

- In C:

  ```c
  int rank;
  MPI_Comm_rank(MPI_COMM_WORLD, &rank);
  ```

- In Fortran (Fortran 90 style):

  ```fortran
  integer :: rank, ierr
  call MPI_Comm_rank(MPI_COMM_WORLD, rank, ierr)
  ```

A **rank** in a communicator is always unique within that communicator. With a unique rank in `MPI_COMM_WORLD`, we can uniquely identify each process in the MPI application.

### Multiple communicators

Initially, all processes started by the launch mechanism are indistinguishable to themselves. A process has to discover:

- Which other processes exist, and
- How to address them.

MPI communicators provide this addressing transparently. Each communicator:

- Numbers its processes from \(0\) to \(n-1\), assuming \(n\) processes in the communicator.
- Allows the *same* MPI process to have *different* ranks in *different* communicators.

This flexibility is important for structuring larger applications, but the basic idea is that communicators define *who can talk to whom* and under which context.


## MPI as a Library: Compiler Wrappers

As we have learned, what we commonly call “MPI” consists of libraries that implement a standardized interface. Including headers (in C) or modules (in Fortran) is only part of the story; when compiling and linking, all required include paths and libraries must be specified correctly.

Because manually specifying all compiler and linker flags can be tedious and error-prone, most MPI implementations provide **compiler wrappers**.

### Compiler wrappers

Compiler wrappers are usually simple shell scripts that:

- Call the underlying compiler (e.g., `gcc`, `icc`, `ifort`).
- Add all the necessary compiler and linker options to find MPI headers, modules, and libraries.
- Accept the usual compiler options from the user.

Common examples include:

- `mpicc` – C compiler wrapper
- `mpicxx` or `mpiCC` – C++ compiler wrapper
- `mpif90` or `mpifort` – Fortran compiler wrappers

The names of these wrappers are **not standardized**, which can be confusing. Some systems may even alias standard compiler commands (like `cc` or `gcc`) to MPI compiler wrappers, so that MPI flags are added automatically.

### Example: environment modules and wrappers

On many HPC systems (for example, the RWTH Aachen compute cluster), environment modules are used. Loading a particular MPI module may:

- Set environment variables (e.g. `CC`, `CXX`, `FC`) to point to the appropriate MPI compiler wrappers.
- Change which compiler and MPI implementation are used.

To see which underlying compiler and flags a wrapper uses, a common pattern is:

```bash
$ mpicc -show
```

This typically prints the full compile and link line that `mpicc` would execute, instead of actually compiling.

From this output you can, for example, infer that:

- `mpicc` uses a particular vendor compiler (e.g. Intel or GNU), and
- It is linked against a specific MPI implementation (e.g. Open MPI or Intel MPI).

On one system, `mpicc` might refer to the Open MPI C wrapper, while after switching modules, `mpiicc` might be the Intel MPI C wrapper. Using the “wrong” wrapper (for example, `mpicc` instead of `mpiicc`) could result in building with a different compiler than you intended.

Because this behavior is not standardized, always consult the documentation of your specific HPC system.


## Launching MPI Applications

After compiling an MPI application, we also need to start it. MPI programs typically consist of multiple processes, potentially distributed across several nodes.

Most MPI implementations provide a launcher program. The standard name in the MPI standard is `mpiexec`. A common invocation looks like:

```bash
mpiexec -n <nprocs> ./program arg1 arg2 ...
```

This starts `<nprocs>` instances of `program` and passes the command-line arguments `arg1`, `arg2`, and so on, to each of them.

### Launchers and resource managers

On real HPC systems, life is usually more complicated:

- The MPI standard specifies `mpiexec`, but does **not** require a particular implementation or behavior.
- Large HPC systems use resource managers and job schedulers (e.g. SLURM, PBS, LSF) to manage parallel jobs for multiple users.

Such systems often require you to use **scheduler-specific** commands to launch an MPI application, for example:

- `runjob` on some IBM Blue Gene/Q systems,
- `srun` with the SLURM resource manager.

To add to the confusion:

- The `mpiexec` (or `mpirun`) command from the underlying MPI library may still be installed and accessible, but using it directly might fail or bypass the scheduler, leading to incorrect or failed runs.

Again, you should always consult the documentation of the HPC system you are using to learn the correct way to launch MPI jobs.

### Responsibilities of the launcher

Launching an MPI job usually requires several things to be set up so the job executes as expected:

1. **Process setup and communication**  
   The launcher ensures that all processes can find each other and establish their initial connections in `MPI_COMM_WORLD`.

2. **Standard output handling**  
   Standard output (`stdout`) from all ranks is typically redirected to your terminal or to the log file of your batch script.  
   Note that this output is **not coordinated**: there are no guarantees on the ordering of lines from different ranks.

3. **Standard input handling**  
   Terminal input (`stdin`) is usually forwarded only to rank 0.

4. **Signal forwarding** (Unix-like systems)  
   Signals sent to the launcher process (e.g. `SIGINT` from `Ctrl+C`) are forwarded to the MPI processes.


## Error Handling in MPI

In our initial examples, we saw that MPI calls return an error code. This allows for runtime failure detection. However, the way errors are handled in MPI is governed by **error handlers** that can be set by the user.

For any non-I/O-related MPI call, the default error handler is:

- `MPI_ERRORS_ARE_FATAL`

With this handler, an error in an MPI call causes the entire MPI job to abort.

This has an important consequence:

- If your application does **not** explicitly change the error handler (for example, to `MPI_ERRORS_RETURN`), then any error-handling code you write *after* a failing MPI call will never be executed, because the program will not return from the call.

In other words, for non-I/O calls, meaningful custom error handling requires you to set a non-fatal error handler first.


## Opaque Objects and Handles

MPI provides many abstractions and hides implementation and platform specifics behind **opaque objects**. Users interact with these objects via **handles**.

Examples include:

- Communicators (`MPI_Comm`)
- Datatypes (`MPI_Datatype`)
- Groups (`MPI_Group`)
- Requests (`MPI_Request`)
- And several others

Key points about handles:

- A handle is a process-local value that refers to some internal MPI object.
- The *numerical value* of a given handle may differ between processes.
- In C, you gain access to handle types by including `mpi.h`.
- In Fortran 90 with `use mpi`, all handle types are `integer`; the compiler cannot check whether you pass the correct handle type to an MPI routine, which can be error-prone when multiple handles appear in a call.
- In Fortran 2008 with `use mpi_f08`, handle types are wrapped in derived types, which enables type checking for procedure arguments.


## MPI Datatypes

Alongside communicators, **datatype handles** are among the most frequently used MPI handles.

### Why datatype abstractions?

MPI introduces datatypes for several reasons:

- MPI provides *transparent marshaling* in heterogeneous environments.  
  This means MPI can take care of different bit representations of basic types on different machines and can convert between them if necessary.

- As a *library*, MPI cannot infer the types and layouts of user buffers at runtime. Therefore, this information must be provided explicitly via datatype handles.

A datatype handle tells MPI:

- How to interpret the memory layout of the data in the buffer.
- How to apply the correct alignment.
- How to convert between different machine representations (if needed).

Variables of type `MPI_Datatype` (in C) are **handles**, not the types themselves. Therefore:

- `sizeof(MPI_Datatype)` yields the size of the handle, *not* the size of the data format it describes.

### Type signature vs. type map

Two important terms in the context of MPI datatypes are:

- **Type signature**  
  A sequence of *basic MPI datatypes* in a buffer, ignoring their exact locations.  
  For each basic MPI datatype, there is a corresponding native language type (e.g. C `int` <-> `MPI_INT`).

- **Type map**  
  A sequence of basic MPI datatypes *together with their locations* in a buffer.

For most communication operations, the **type signatures** on the sending and receiving sides must match, whereas the **type maps** may differ across processes.

### Basic MPI datatypes

MPI provides a set of basic datatypes corresponding to common native language types, such as:

- Integers
- Floating-point numbers
- Logical/boolean types
- Characters

In addition, there is a special datatype handle:

- `MPI_BYTE`  
  Represents a sequence of 8 binary digits (a byte) and can be used for *untyped* data. For `MPI_BYTE`, no datatype conversion is performed.


## Blocking and Non-blocking, Synchronous and Asynchronous

Many MPI procedures and operations are defined as *blocking* or *non-blocking*, and the terminology can be a source of confusion.

### Blocking operations

A **blocking** MPI procedure returns when the associated operation is **complete locally**. This typically means:

- Any buffers passed to the call may be safely reused or deallocated by the user once the call returns.

However, the return from a blocking function usually does **not** imply anything about the completion of the operation on the remote side.

### Non-blocking operations

A **non-blocking** MPI procedure returns **before** the associated operation is complete locally. This implies:

- At least one additional MPI call (e.g. `MPI_Wait` or `MPI_Test`) is needed to complete the pending operation.
- While the operation is not complete, the user must **not** modify any arguments (such as buffers) associated with that operation.

Non-blocking operations allow for overlapping communication with computation, but they require careful management of pending operations and buffers.

### Synchronous vs. asynchronous

Non-blocking operations are sometimes informally referred to as “asynchronous,” but in MPI terminology, these are distinct concepts:

- A call is **synchronous** (or synchronizing) if it can complete locally *only with explicit intervention of the remote communication partner*.  
  Example: certain synchronous send operations.

- A call is **asynchronous** if it can complete locally without any explicit intervention from the remote side.

The term **non-blocking** only describes *when the call returns* relative to the completion of the local operation. It does **not** state whether the operation is synchronous or asynchronous.


## Communication Paradigms in MPI

MPI provides three main communication paradigms.

### 1. Point-to-point communication

Point-to-point communication is a direct data exchange between **two** processes:

- One process explicitly **sends** data.
- The other process explicitly **receives** data.

Point-to-point operations are the building blocks for many MPI communication patterns.

### 2. Collective communication

Collective communication involves a **group of processes**, typically all processes in a communicator. Examples include:

- Broadcast (one-to-many)
- Reduction (many-to-one)
- Scatter and gather

A real-world analogy is a lecture or radio broadcast where a single source sends information to many recipients. Other patterns (such as many-to-one or all-to-all) are also possible, but the key characteristic is that the whole group participates in the communication operation.

### 3. One-sided communication

One-sided communication is a data exchange between two processes where **one process specifies all communication parameters**, including:

- Where in a predefined memory window the data is placed on the remote side,
- How much data is transferred,
- The direction of transfer (put/get).

The remote process does not explicitly participate in the individual communication operation; instead, it exposes a memory window that the originating process accesses.


## Summary

This episode provided a brief introduction to several key MPI concepts:

- MPI is a **library**, not a language extension, and therefore must be explicitly initialized with `MPI_Init` and finalized with `MPI_Finalize`, each called exactly once.
- Communicators such as `MPI_COMM_WORLD` define communication contexts. Within a communicator, each process has a unique **rank**, and the number of processes can be queried dynamically.
- **Compiler wrappers** (e.g. `mpicc`, `mpiicc`) and system-specific **launchers** (e.g. `mpiexec`, `srun`) help compile and run MPI applications on HPC systems.
- MPI uses **opaque handles** for objects such as communicators and datatypes; datatype handles describe how to interpret and transfer data.
- MPI distinguishes between **blocking** and **non-blocking** operations, and also between **synchronous** and **asynchronous** behavior; these notions are related but not identical.
- MPI supports three main communication paradigms: **point-to-point**, **collective**, and **one-sided** communication.

We will explore many of these concepts in more depth in later parts of this tutorial.

:::::::::::::::::::::::::::::::::::::: keypoints

- MPI programs must call `MPI_Init` and `MPI_Finalize` exactly once.
- `MPI_COMM_WORLD` contains all initially started processes; ranks are unique within a communicator.
- Use MPI compiler wrappers and system-specific launch commands to compile and launch MPI programs.
- MPI objects are accessed via opaque handles; datatype handles describe the layout and conversion of communicated data.
- Blocking vs. non-blocking and synchronous vs. asynchronous are distinct concepts in MPI.
- MPI provides point-to-point, collective, and one-sided communication paradigms.

::::::::::::::::::::::::::::::::::::::::::::::::
