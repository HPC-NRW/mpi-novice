---
title: "Blocking Point-to-Point Communication"
teaching: 30
exercises: 15
---

::::::: questions
- How do two MPI processes exchange data using blocking point-to-point communication?
- What information is required to match a send with a receive?
- How do different MPI send modes influence performance and deadlock behavior?
:::::::::::::::::

:::::: objectives
- Explain the semantics and arguments of `MPI_Send` and `MPI_Recv`.
- Describe how MPI matches messages using the message envelope and type signature.
- Use `MPI_Status`, `MPI_Get_count`, and `MPI_Probe` to handle messages of unknown size.
- Recognize and avoid deadlocks with blocking communication.
- Use `MPI_Sendrecv` and `MPI_Sendrecv_replace` for safe bidirectional communication.
:::::::::::::::::

## Introduction

In this episode we look at **blocking point-to-point communication** in MPI.

Point-to-point communication is the explicit exchange of data between **exactly two**
processes: one **sender** and one **receiver**. You can think of it as one process
“reporting” some data to a specific partner.

We assume the usual SPMD (Single Program, Multiple Data) setup:

- The same program runs on all MPI processes.
- Processes are identified by their **rank** in a communicator (typically `MPI_COMM_WORLD`).
- Each process can send data to, and receive data from, other ranks.

Blocking point-to-point communication in MPI is mainly implemented through:

- `MPI_Send` – blocking send
- `MPI_Recv` – blocking receive

These operations complete only when MPI considers the operation **locally complete**.
We will see what that means in detail.


## The basic blocking operations: `MPI_Send` and `MPI_Recv`

The most fundamental point-to-point operations in MPI are the blocking send and receive.

In C, their prototypes (simplified) are:

```c
int MPI_Send(
    const void *buf, int count, MPI_Datatype datatype,
    int dest, int tag, MPI_Comm comm
);

int MPI_Recv(
    void *buf, int count, MPI_Datatype datatype,
    int source, int tag, MPI_Comm comm, MPI_Status *status
);
```

In Fortran, the argument list is similar (with an additional `ierror` argument).

### Data description: `buf`, `count`, `datatype`

The first three arguments describe **what** is sent or received:

- `buf`  
  Pointer (C) or array (Fortran) to the start of the send/receive buffer.

- `count`  
  Number of elements to be sent or received.

- `datatype`  
  MPI datatype describing the type of the buffer elements.

Together, `count` and `datatype` form the **type signature** of the message. Conceptually,
the signature is

\[
\text{signature} = \text{count} \times \text{signature of datatype}.
\]

This does **not** necessarily mean that sender and receiver must use the same native
language type; MPI datatypes allow you to build more complex layouts. However, the **overall
type signature must be compatible** between sender and receiver for correct communication.

MPI also provides **transparent type marshaling** in heterogeneous environments: the
`datatype` tells MPI how to pack and unpack the data correctly across different
architectures.

### Communication partner and context: `dest` / `source`, `tag`, `comm`

The next three arguments specify **with whom** the process communicates and in which
context:

- `dest` (in `MPI_Send`)  
  Rank of the receiving process in communicator `comm`.

- `source` (in `MPI_Recv`)  
  Rank of the sending process in communicator `comm`.

- `tag`  
  An integer used to label messages. Tags allow you to distinguish different kinds
  of messages between the same pair of processes.

- `comm`  
  Communicator that defines the group of processes and the communication context.

Tags are specific to point-to-point communication; you do not see them in collective
operations.

On the receive side, `source` and `tag` can use special wildcard values:

- `MPI_ANY_SOURCE` – match a message from any rank in `comm`.
- `MPI_ANY_TAG` – match a message with any tag.

The actual source and tag of the received message are then reported in the
**status object**.


## The message envelope and message matching

For correct communication, **every send must be matched by exactly one receive**, and
vice versa. MPI uses the **message envelope** to determine which send matches which
receive.

The envelope consists of:

- `source` rank
- `destination` rank
- `tag`
- `communicator` `comm`

On the sending side, `source` is implicit (it is the rank calling `MPI_Send`),
and `dest` is explicit. On the receiving side, `source` is explicit and `dest` is implicit.

A receive matches a send if:

1. The communicator is the same (`comm`).
2. The source matches (`source` equals the sender rank, or `MPI_ANY_SOURCE`).
3. The tag matches (`tag` equals the send tag, or `MPI_ANY_TAG`).
4. The **type signature** (described by `count` and `datatype`) is compatible.

Note:

- Messages do not **aggregate**: one receive matches one send.
- Messages do not **split**: you cannot receive half a message with one call and the other half with another call.

::: callout
### Type compatibility is your responsibility

A mismatch in `count` and `datatype` between sender and receiver might **not** be detected
by the MPI implementation. This can lead to subtle and hard-to-debug errors. MPI assumes
you pass compatible type signatures.
:::


## Buffer sizes and `MPI_Get_count`

The receive buffer must be large enough to hold the entire message:

- It is **allowed** to have a **larger receive buffer** than the message size.
- It is **erroneous** (and typically fatal) to have a **smaller receive buffer** than the message size:
  the message is truncated, and the default error handler will usually abort the program.

In many applications you may not know the message size in advance. However, if you
use a receive buffer large enough to cover the **maximum possible message size**, you can
later query the actual size.

### Querying the received size: `MPI_Get_count`

After `MPI_Recv` returns, you can use `MPI_Get_count` on the associated `status`
to determine how many elements were actually received:

```c
int MPI_Get_count(
    const MPI_Status *status,
    MPI_Datatype datatype,
    int *count
);
```

- You provide the `datatype` you want to interpret the message with.
- MPI checks whether the message size is an integer multiple of that datatype.
  - If yes, `count` is set to the number of elements.
  - If not, `count` is set to `MPI_UNDEFINED`.

This is particularly useful when:

- Messages have variable sizes.
- You receive into a buffer sized for a known maximum.
- You want to process exactly the received part, not the whole buffer.


## The `MPI_Status` object

The `status` argument in `MPI_Recv` is an opaque MPI type that describes the outcome of
the receive operation.

Conceptually, it contains at least:

- `MPI_SOURCE` – the actual source rank of the message.
- `MPI_TAG` – the actual tag of the message.
- `MPI_ERROR` – an error code for this operation.

In C, you typically access these as:

```c
status.MPI_SOURCE
status.MPI_TAG
status.MPI_ERROR
```

In Fortran, the `STATUS` array has predefined indices such as `MPI_SOURCE` and `MPI_TAG`.

This is especially important if you used wildcards (`MPI_ANY_SOURCE`, `MPI_ANY_TAG`) in the receive:
the `status` object tells you which message you actually got.

The error code is usually not inspected when using the default error handler
(`MPI_ERRORS_ARE_FATAL`), because in that case the program aborts on error. If you change
the error handler to `MPI_ERRORS_RETURN`, you can check `status.MPI_ERROR`.


## Probing for message size: `MPI_Probe`

Sometimes you **cannot** easily choose a safe maximum message size. In that case, you can
use `MPI_Probe` to determine the size of a pending message before actually receiving it.

```c
int MPI_Probe(
    int source, int tag, MPI_Comm comm, MPI_Status *status
);
```

`MPI_Probe`:

- Is **blocking**: it waits until a matching message is available.
- Does **not** receive any data.
- Fills the `status` object as if you had called `MPI_Recv`.

Typical usage:

1. Call `MPI_Probe(source, tag, comm, &status)` to wait for a message matching the envelope.
2. Use `MPI_Get_count(&status, datatype, &count)` to determine the message size.
3. Allocate a buffer of the appropriate size.
4. Call `MPI_Recv(buf, count, datatype, source, tag, comm, &status)` to actually receive the data.

You may still use wildcards (`MPI_ANY_SOURCE`, `MPI_ANY_TAG`) in `MPI_Probe`.

::: callout
### Important: matching `MPI_Probe` and `MPI_Recv`

If you probed with wildcards, you **must not** use wildcards again in the corresponding
`MPI_Recv`. Instead, use the specific `source` and `tag` from the `status` returned
by `MPI_Probe`.

Otherwise, you might accidentally receive a **different** message that happens to match
the same wildcard envelope, especially when multiple messages are pending.
:::

### Incorrect pattern

```c
MPI_Status status;
MPI_Probe(MPI_ANY_SOURCE, MPI_ANY_TAG, comm, &status);

/* WRONG: using wildcards again */
MPI_Recv(buf, count, datatype, MPI_ANY_SOURCE, MPI_ANY_TAG, comm, &status);
```

::: warning
This is erroneous and leads to **non-deterministic behavior**: whether it fails or not
depends on which messages are available at the time of the receive.
:::::::::::

### Correct pattern

```c
MPI_Status status;
MPI_Probe(MPI_ANY_SOURCE, MPI_ANY_TAG, comm, &status);

int source = status.MPI_SOURCE;
int tag    = status.MPI_TAG;
int count;
MPI_Get_count(&status, datatype, &count);

MPI_Recv(buf, count, datatype, source, tag, comm, &status);
```

Here the receive is guaranteed to match the same message that was probed.


## Local completion of blocking operations

MPI defines **completion** from the perspective of the calling process.

An operation is **locally complete** when:

- For a **send**, the user’s send buffer is no longer needed by MPI.
- For a **receive**, the receive buffer now contains the complete message.

This has important implications:

- For **receives**, completion is intuitive: the blocking `MPI_Recv` returns only when
  the full message has arrived and has been stored in the receive buffer.

- For **sends**, completion is more subtle:
  - The send may be complete **even if the data has not yet left the node**.
  - For example, if MPI copies the data into an internal buffer, it can return control
    to the user while the actual network transfer happens later in the background.

From the application’s perspective:

- Once a blocking send or receive returns, you may **safely reuse or modify** the buffer
  passed to MPI.
- However, you do **not** get any guarantee about the state of the remote buffer or the
  remote process. Blocking sends do **not** imply remote completion.


## Send modes: standard, buffered, synchronous, ready

MPI provides different send operations with different semantics:

- **Standard send**: `MPI_Send`
- **Buffered send**: `MPI_Bsend`
- **Synchronous send**: `MPI_Ssend`
- **Ready send**: `MPI_Rsend`

All of these are **blocking** in the sense that they only return after **local completion**.
The difference lies in when and how MPI is allowed to buffer data and when it waits for
the matching receive.

### Standard send: `MPI_Send`

`MPI_Send` is the most commonly used send operation. Its behavior is **implementation dependent**:

- For **small messages**, MPI often uses an **eager** protocol:
  - The message is buffered internally and sent immediately.
  - `MPI_Send` may return before the receive has been posted.

- For **large messages**, MPI often uses a **rendezvous** protocol:
  - The sender first sends only the envelope.
  - It waits until the receiver has posted a matching receive.
  - Only then is the bulk data transferred.

The switch between eager and rendezvous is transparent to the user, which can lead to
surprising deadlocks if you rely on eager behavior.

### Buffered send: `MPI_Bsend`

`MPI_Bsend` explicitly uses user-provided buffer space:

- You must attach a buffer with `MPI_Buffer_attach` before using `MPI_Bsend`.
- MPI copies data from your send buffer into this attached buffer, then returns.
- The actual network transfer can happen later.

This **forces an extra copy**, which can be expensive. It is rarely necessary for
most applications, because standard sends typically already use internal buffering
for small messages.

### Synchronous send: `MPI_Ssend`

`MPI_Ssend` implements an explicit **rendezvous** send:

- The call returns at the earliest when the matching receive has been **started** on
  the receiver.
- It may still return before the entire message has been received, if MPI internally
  buffers the remainder.

Use `MPI_Ssend` when you want to enforce a synchronization between sender and receiver:
both sides must participate before either continues.

### Ready send: `MPI_Rsend`

`MPI_Rsend` assumes that the matching receive has already been posted:

- If the receive is **not** posted, the behavior is erroneous and typically aborts.
- The theoretical motivation is that MPI can skip some safety checks and save time.
- In practice, it is difficult to guarantee that the receive is already posted, and
  the potential performance gain is small.

The common recommendation is **to avoid `MPI_Rsend`**.


## Deadlocks with blocking sends and receives

Blocking communication can easily lead to deadlocks if the communication pattern
creates cyclic dependencies.

Consider two ranks exchanging data:

```c
/* Rank 0 */
MPI_Send(sendbuf0, count, datatype, 1, 0, comm);
MPI_Recv(recvbuf0, count, datatype, 1, 0, comm, &status);

/* Rank 1 */
MPI_Send(sendbuf1, count, datatype, 0, 0, comm);
MPI_Recv(recvbuf1, count, datatype, 0, 0, comm, &status);
```

What happens?

- If both sends are implemented using the **eager** protocol (small messages):
  - Both `MPI_Send` calls may complete because MPI buffers the data internally.
  - Then both `MPI_Recv` calls complete.
  - This program appears to work.

- If the messages are large and `MPI_Send` uses the **rendezvous** protocol:
  - Rank 0 waits in `MPI_Send` for rank 1 to post the matching receive.
  - Rank 1 waits in `MPI_Send` for rank 0 to post the matching receive.
  - Neither process posts a receive; both are stuck forever → **deadlock**.

This is particularly confusing because the same code may work for small messages and
deadlock for larger ones, depending on the MPI implementation and configuration.

::: callout
### Never rely on implementation-dependent eager behavior

Always design your communication patterns so that they are safe even if `MPI_Send`
uses the rendezvous protocol (i.e., behaves like a synchronous send).
:::

### Fixing the pattern: receive-first on one rank

One simple fix is to make one rank receive first:

```c
/* Rank 0 */
MPI_Recv(recvbuf0, count, datatype, 1, 0, comm, &status);
MPI_Send(sendbuf0, count, datatype, 1, 0, comm);

/* Rank 1 */
MPI_Send(sendbuf1, count, datatype, 0, 0, comm);
MPI_Recv(recvbuf1, count, datatype, 0, 0, comm, &status);
```

Now there is no cyclic dependency:

- Rank 0 posts the receive and can accept data from rank 1.
- Rank 1 sends to rank 0, which succeeds.
- Both then perform the second operation without deadlock.

The downside is that you now have **asymmetric code**: some ranks “receive-first”, others
“send-first”. This can make code less readable and harder to maintain.


## `MPI_Sendrecv` and `MPI_Sendrecv_replace`

A more structured way to perform a **two-way exchange** without deadlock is to use
`MPI_Sendrecv`.

```c
int MPI_Sendrecv(
    const void *sendbuf, int sendcount, MPI_Datatype sendtype,
    int dest, int sendtag,
    void *recvbuf, int recvcount, MPI_Datatype recvtype,
    int source, int recvtag,
    MPI_Comm comm, MPI_Status *status
);
```

`MPI_Sendrecv` conceptually combines one send and one receive into a single call:

- It takes all the arguments you would pass to separate `MPI_Send` and `MPI_Recv`.
- It ensures that the communication completes without deadlock when used symmetrically.

Example: pairwise exchange between two processes:

```c
/* Both Rank 0 and Rank 1 execute: */
MPI_Sendrecv(
    sendbuf, count, datatype, partner_rank, 0,
    recvbuf, count, datatype, partner_rank, 0,
    comm, &status
);
```

MPI internally arranges the order of sending and receiving so that there is no cyclic
dependency between the two processes.

### `MPI_Sendrecv_replace`

Sometimes you want the exchange to happen **in-place**, i.e., use the same buffer for
sending and receiving. Then you can use:

```c
int MPI_Sendrecv_replace(
    void *buf, int count, MPI_Datatype datatype,
    int dest, int sendtag,
    int source, int recvtag,
    MPI_Comm comm, MPI_Status *status
);
```

Here:

- `buf` serves as both send and receive buffer.
- `count` and `datatype` are shared for both directions.
- This reduces the number of arguments, but also imposes stricter semantics,
  and can be slightly less efficient.


## Message ordering

MPI guarantees **non-overtaking** for messages between the same pair of processes
within the same communicator:

- If a process sends multiple messages to the same destination, and the receiver posts
  matching receives, the messages will be received in the **same order** they were sent.
- This also applies when using `MPI_Probe`: probes will observe messages in order.

This guarantee only holds for:

- Messages between the **same source and destination**.
- Messages in the **same communicator**.

It does **not** necessarily hold:

- Across **different communicators** (even between the same ranks).
- For messages originating from **different senders**.
- When **multiple threads** on the sending process send to the same receiver on the same
  communicator — these sends are considered logically concurrent.

Ordering guarantees can be useful, for example, when:

- You send several messages with `MPI_Send` (standard mode).
- You send the **last** one with `MPI_Ssend` (synchronous).
- When the synchronous send returns, you know that all previous messages have at least
  started to be received, because they cannot overtake the last one.


::: challenge
### Challenge: Thinking about buffer sizes

Suppose process 0 wants to send a vector of `double` values of size `n` to process 1.
Process 1 does **not** know `n` in advance, but knows that it is at most 1000.
Sketch how process 1 can safely receive the data using blocking communication.

- How would you do it **without** `MPI_Probe`?
- How would you do it **with** `MPI_Probe` and `MPI_Get_count`?

:::

::: solution
**Without `MPI_Probe`:**

- Process 1 allocates a buffer for 1000 doubles (the maximum possible size).
- It calls `MPI_Recv(buf, 1000, MPI_DOUBLE, 0, tag, comm, &status)`.
- After the receive, it calls `MPI_Get_count(&status, MPI_DOUBLE, &n_received)`
  to find out how many elements were actually sent.

**With `MPI_Probe`:**

- Process 1 calls `MPI_Probe(0, tag, comm, &status)`.
- It calls `MPI_Get_count(&status, MPI_DOUBLE, &n)` to determine the exact size.
- It allocates a buffer of size `n`.
- It calls `MPI_Recv(buf, n, MPI_DOUBLE, 0, tag, comm, &status)` to actually receive
  the data.
:::


::::::: keypoints
- MPI provides blocking point-to-point communication via `MPI_Send` and `MPI_Recv`.
- Messages are matched using the **message envelope**: source, destination, tag, and communicator, plus a compatible **type signature** (count × datatype).
- The receive buffer must be large enough; use `MPI_Get_count` and/or `MPI_Probe` to handle messages of unknown size.
- Blocking operations complete **locally**: send buffers can be reused after completion, but completion does not imply that the data has already arrived at the remote side.
- Different send modes (`MPI_Send`, `MPI_Bsend`, `MPI_Ssend`, `MPI_Rsend`) influence buffering and synchronization; in practice, use **standard send** by default and **synchronous send** when you need explicit synchronization.
- Do not rely on eager buffering to avoid deadlock. Design communication patterns that are safe even when `MPI_Send` behaves like a rendezvous send.
- `MPI_Sendrecv` and `MPI_Sendrecv_replace` provide a convenient way to implement safe bidirectional communication in a single call.
- Message ordering is preserved between the same pair of processes in the same communicator, but not necessarily across different communicators, different senders, or multiple sending threads.
::::::::::::::::::

