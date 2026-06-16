---
title: "Why Message Passing and MPI?"
teaching: 20
exercises: 10
---

:::::: questions

- How do shared-memory and distributed-memory systems differ?
- Why do we need message passing on distributed-memory machines?
- What is the SPMD (Single Program, Multiple Data) execution model?
- What is MPI and what role does it play in HPC applications?

::::::::::::::::

:::::: objectives

- Describe the basic architecture of distributed-memory HPC systems.
- Contrast shared-memory and distributed-memory programming models.
- Explain why explicit data exchange and data replication are central to message passing.
- Describe what an operating system process is and why processes are isolated.
- Explain the SPMD model and list key requirements of SPMD-style parallel programs.
- Summarize what MPI is, what it is not, and how it is provided on HPC systems.

:::::::::::::::::

## Distributed-Memory HPC Systems

High-performance computing (HPC) systems today are dominated by **distributed-memory multi-computers**.

- An HPC system consists of many smaller computers, often called **nodes**.
- Each node:
  - has its own local memory,
  - usually runs its own instance of an operating system (often a stripped-down variant compared to a desktop OS),
  - is connected to other nodes by a **high-bandwidth, low-latency network**.

In such systems, **one node cannot directly access the memory of another node**. This is what we mean by *distributed memory*.

To solve a common computational problem efficiently on these systems, we must coordinate many processes running on different nodes. One widely used approach is **message passing**, where processes explicitly send and receive data via the network.

## Shared vs. Distributed Memory Programming

Before looking at message passing, it is useful to contrast it with **shared-memory** programming.

### Shared-Memory Model

In an abstract shared-memory machine:

- Multiple processing elements (cores or hardware threads) are connected to a **single shared memory**.
- All processing elements can directly load from and store to the same memory locations.

Modern multicore CPUs and even smartphones are examples:

- A single process can have multiple threads.
- All threads share one address space and can access shared variables.

To exchange data between two processing elements:

- One writes the data to a shared variable in memory.
- The other reads that data from the same variable.

Shared-memory programming involves:

- *Declaring shared variables* that multiple threads can access.
- *Synchronizing accesses* (e.g., with locks, barriers, or atomics) to avoid **data races**.

These details are outside the scope of this MPI-focused lesson, but technologies like **OpenMP** are commonly used for shared-memory parallel programming.

### Distributed-Memory Model

In a distributed-memory scenario:

- Each processing element (or process) has access only to its **local memory**.
- To exchange data:
  - Process A must **send** data from its local memory over the network.
  - Process B must **receive** this data and store its own **local copy**.

Important consequences:

- **Data replication**: once Process B receives data from Process A, there are now two copies (one in each local memory).
- If many processes share the same data, this replication can significantly increase the **overall memory footprint** and can limit scalability.

::: challenge
### Concept Check: Shared vs Distributed Memory

On a shared-memory system, two threads can access the same array in memory.  
On a distributed-memory system, two processes each have their own copy of an array.

1. How can two threads on a shared-memory system share data?
2. How must two processes on a distributed-memory system share data?

::: solution

1. On a shared-memory system, two threads can share data by reading and writing the same variable in the shared address space, using synchronization (locks, barriers, etc.) to avoid data races.

2. On a distributed-memory system, each process has its own address space. To share data, one process must send the data and the other must receive it (e.g., using MPI), creating a separate copy in the receiver’s local memory.
:::
:::

## Explicit vs Implicit Message Passing

In distributed-memory programming, data must be **explicitly exchanged** between processing elements.

There are two main views:

- **Explicit message passing**:
  - The application calls communication procedures directly (e.g., `MPI_Send`, `MPI_Recv`).
  - The programmer controls when, what, and where data is sent and received.

- **Implicit message passing**:
  - Some languages and models (e.g., PGAS — Partitioned Global Address Space) present a *shared-memory abstraction*.
  - The runtime system automatically performs data transfers in the background when you access remote data.

In both cases, there are **no truly shared variables** across processes: all variables are local to a process. This also means:

- There are **no data races** of the kind seen in shared-memory programming, because two processes cannot accidentally write to the same memory location.
- Data exchange implies some level of **implicit synchronization**:
  - A receive operation can only complete once the corresponding send operation has started (and in some modes, completed).
  - This provides a natural synchronization point between processes.

## Processes and Operating System Isolation

To understand message passing, it helps to understand what an **operating system process** is.

- An **executable** (or binary) is a file containing machine instructions.
- A **process** is a running, in-memory instance of an executable.

A process typically has:

- One or more **threads of execution**:
  - Threads share the process’s memory.
  - Threads may also have some local storage that other threads cannot access.
- Several types of memory:
  - **Heap**: dynamically allocated memory (e.g., via `malloc` in C) that can live across function calls.
  - **Stack**: memory used for function-local variables and call frames; automatically allocated and freed when functions are entered and left.
  - **Registers**: CPU storage for very fast access. Most scientific applications rely on the compiler to manage registers.

A process also has **operating system context**, such as:

- Open file handles.
- Signal handlers.
- A process identifier (**PID**) used by the OS to track the process.

### Isolation and Protection

A key principle of processes is **isolation**:

- One process cannot (and should not) directly access the memory of another process.
- This prevents faulty or malicious processes from corrupting other processes.

Consequences:

- For a process to:
  - communicate with another process, or
  - interact with external resources (files, network, etc.),
  
  it must go through the **operating system**, typically via **system calls**.
  
- There are no direct means for:
  - inter-process data exchange, or
  - inter-process synchronization,
  
  without OS or library support.

Message passing libraries such as MPI are built on top of these capabilities and provide a convenient **programming interface for inter-process communication**.

## The SPMD Execution Model

One of the main parallel execution models used in HPC is **SPMD**:

- **S**ingle **P**rogram, **M**ultiple **D**ata.

Characteristics:

- A single executable is run in parallel by many instances (processes or threads).
- Every instance executes the *same program*, but:
  - has its own **instruction flow**,
  - has its own **identifier** (rank, thread ID, etc.),
  - typically works on a **different part of the data**.

This differs from **MPMD** (Multiple Programs, Multiple Data):

- Different executables cooperate to achieve a common goal.
- Example: a web browser and a web server working together.

In SPMD:

- Each instance can:
  - choose different code paths based on its ID,
  - handle different subdomains of a larger dataset,
  - emulate MPMD behavior by branching into different parts of the code.

SPMD is not the only execution model in HPC, but it is currently the **predominant model** for MPI-based applications.

### SPMD in Practice

A typical workflow looks like this:

1. **Compile and link** source code using a compiler (plus a linker) to create an executable.
2. **Launch multiple processes** running this same executable, often across many nodes.
3. Each process:
   - determines its own ID,
   - selects the appropriate subset of the overall data to process.
4. At some point (often at the end of the program), processes **combine their local results** into a global result, for example by gathering data to one process or performing a reduction.

---

::: challenge
### Concept Check: SPMD

Consider an SPMD program that approximates an integral by splitting the integration interval into sub-intervals.

1. How can different processes share the work?
2. Why is it useful for each process to know its own ID (rank)?

::: solution

1. Each process can work on a different sub-interval of the integration domain. For example, if there are \(N\) processes and an interval \([a, b]\), process \(i\) can handle a sub-interval that depends on \(i\).

2. The process ID (rank) lets each process decide which part of the data it is responsible for. Using its rank, a process can compute its sub-interval bounds or array indices and avoid overlapping with other processes.
:::
:::

## Requirements for SPMD Environments

To enable effective SPMD-style parallel programming, we need several core capabilities:

1. **Identification of all participants**

   - Each process (or thread) must be able to answer:
     - “Who am I?” (its own ID, e.g. rank).
     - “Who else is participating?” (the size of the group and the IDs of peers).

2. **Robust data exchange mechanisms**

   - We must be able to:
     - specify where data is going to (destination ID) or coming from (source ID),
     - describe the layout, size, and type of the data,
     - know when the data is **safe to use** (i.e., the communication has completed).

3. **Synchronization of instruction flow**

   - Processes must be able to:
     - coordinate their progress,
     - wait until certain communication or computation steps are finished before proceeding.

4. **Launching processes (possibly across nodes)**

   - There must be a robust mechanism to start and manage multiple processes, often on different nodes in an HPC cluster.

5. **Portability across platforms**

   - It is highly desirable that the same parallel program can run on different HPC systems with minimal or no modifications, even if those systems have different CPUs or interconnects.

The **Message Passing Interface (MPI)** was developed to address these needs.

## The Message Passing Interface (MPI)

**MPI** is the de facto standard **Application Programming Interface (API)** for explicit message passing in HPC.

Key characteristics:

- MPI is a **standard interface definition**:
  - Specified and maintained by the non-profit **Message Passing Interface Forum**.
  - The standard documents are publicly available at <https://www.mpi-forum.org/>.

- MPI is **not**:
  - a programming language,
  - a compiler extension,
  - or a single specific implementation.

Instead:

- MPI defines:
  - a set of functions and their semantics (send, receive, collectives, communicators, etc.),
  - how these functions should behave, in a language-independent way.
- Implementations of MPI (libraries) must comply with this interface.

### MPI Implementations

Several MPI implementations exist, for example:

- [**Open MPI**](https://open-mpi.org/)
- [**MPICH**](https://mpich.org/)
- [**MVAPICH**](https://mvapich.cse.ohio-state.edu)
- [**Intel MPI**](https://www.intel.com/content/www/us/en/developer/tools/oneapi/mpi-library.html)

::::::: warning
**ABI compatibility** (binary compatibility) was *not* guaranteed between implementations prior to MPI 5.0.
When you are not using an implementation compliant to MPI 5.0 or later, applications usually must be compiled and linked against the specific MPI implementation they will use and all parts of an application must use the same implementation.
:::::::::::::::

### Language Bindings

The MPI standard provides:

- A **language-independent specification**.
- Standardized bindings for:
  - **C**
  - **Fortran**

These remain the predominant languages in HPC.

Historically, MPI defined specific **C++ bindings**, but these have been deprecated and removed from the standard. It is now recommended to:

- call the C bindings from C++, or
- use higher-level abstractions such as:
  - [**Boost.MPI**](https://www.boost.org/doc/libs/latest/doc/html/mpi.html)
  - [**MPL**](https://rabauke.github.io/mpl/html/) (Message Passing Library for C++)

There are also *non-standard* bindings for other languages, for example:

- **Python**: [**mpi4py**](https://mpi4py.readthedocs.io/en/stable/mpi4py.html)
- **Java**: [bindings from Open MPI](https://docs.open-mpi.org/en/v5.0.x/features/java.html) or [MPJ Express](http://www.mpjexpress.org/)

### MPI as a Library

MPI is provided as:

- Libraries (e.g., `libmpi`),
- Header files (e.g., `mpi.h` for C),
- Auxiliary compiler wrapper commands (e.g., `mpicc`, `mpicxx`, `mpif90`).

You still use standard compilers (like `gcc`, `clang`, `icc`, `gfortran`) via these wrappers. MPI is **not** a language extension; it is a set of functions you call from your regular C, C++, Fortran (or other) programs.

## Further Reading and Documentation

To learn more about MPI:

- The **MPI standard documents** (MPI 3.x, MPI 4.x) can be downloaded from the MPI Forum website:
  - <https://www.mpi-forum.org/>
- MPI implementations typically provide **man pages** for individual MPI functions:
  - For example: `man MPI_Send` on many HPC systems.
- There are many **tutorials and training materials** available online, including:
  - Short tutorials (like this one),
  - Longer courses and books dedicated to MPI and parallel programming.

Keep in mind:

- The MPI standard is written for both users and implementers, but it is **not designed as introductory teaching material**.
- Tutorials, books, and courses are often better starting points for learning MPI concepts and best practices.

::::::: keypoints

- Modern HPC systems are typically distributed-memory multi-computers composed of many networked nodes.
- On distributed-memory systems, processes cannot directly access each other’s memory; they must exchange data explicitly, usually via message passing.
- Message passing leads to replication of data and provides implicit synchronization between sender and receiver.
- Processes are isolated operating system entities with their own address space; inter-process communication requires operating system and library support.
- The SPMD (Single Program, Multiple Data) model is the predominant execution model for MPI programs.
- MPI is a *standardized interface* for explicit message passing, with multiple implementations and language bindings; it is a library, not a language extension.

:::::::::::::::::

