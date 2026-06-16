---
title: "Nonblocking Point‑to‑Point Communication"
teaching: 30
exercises: 0
---

::: questions
- What is the fundamental difference between blocking and non-blocking communication in MPI?
- When should you use non-blocking communication versus blocking communication?
- How do request handles facilitate the completion of non-blocking operations?
:::

::: objectives
- Explain the concept of non-blocking communication and its advantages
- Differentiate between blocking completion (wait family) and non-blocking testing (test family)
- Describe the role of request handles in managing non-blocking operations
- Understand local completion semantics and their implications for buffer reuse
- Apply non-blocking techniques to prevent deadlocks in point-to-point communication
:::


## Motivation: Why Non-blocking Communication?

::: instructor
Note: Connect this episode to the previous one where students learned about blocking point-to-point communication. Emphasize that non-blocking is an orthogonal concept that extends all communication modes.
:::

Non-blocking communication is an **orthogonal concept** that extends all the functionality you learned about yesterday in point-to-point communication [transcript_04_PPCES 2025 - Non-blocking Point-to-Point Communication.txt]. For every send mode (buffered, synchronous, standard) and every receive mode, there exists a non-blocking counterpart that separates the initiation of the operation from its completion.

> **Key Insight**: The non-blocking interface provides a mechanism to initiate communication operations that return immediately, allowing your program to continue executing while the communication progresses in the background.

This concept is not limited to point-to-point communication. The same non-blocking principles apply to:
- Collective communication
- Parallel file I/O

The knowledge you gain here will transfer directly to those advanced topics later in your MPI education [transcript_04_PPCES 2025 - Non-blocking Point-to-Point Communication.txt].

## The Non-blocking Communication Model

### Initiation vs. Completion

In non-blocking communication, the fundamental paradigm is:

```python
# Conceptual model
initiate_operation(buffer, ...)  # Returns immediately
# Do useful work here while communication progresses
complete_operation(request)       # Waits for completion if needed
```

::: callout
The defining characteristic of non-blocking procedures is that **they return before the associated operation completes locally**. This means you cannot safely reuse your communication buffers until you have confirmation of local completion.
:::

### Request Handles: The Missing Link

To manage this separation between initiation and completion, MPI introduces **request handles** (of type `MPI_Request` in C):

::: callout
**Request Handle**: A unique identifier that tracks the status of a non-blocking operation. It serves as the connection between the initiation call and the completion call.
:::

- In C: `MPI_Request` type
- In Fortran 90: Integer type
- In Fortran 2008: Type leaning toward the C interface

::: callout
**Important**: The "I" in function names like `MPI_Isend` and `MPI_Irecv` stands for **Incomplete** (though some debate whether it means **Immediate**). This emphasizes that the operation has not finished when the function returns.
:::

## Completion Mechanisms: Wait vs. Test

MPI provides two families of functions to complete non-blocking operations:

### Blocking Completion: The Wait Family

These functions suspend execution until the specified operation(s) complete:

```c
// MPI_Wait - Wait for a single request to complete
MPI_Wait(request, status);

// MPI_Waitany - Wait for any one of several requests to complete
MPI_Waitany(count, array_of_requests, index, status);

// MPI_Waitsome - Wait for some (one or more) requests to complete
MPI_Waitsome(incount, array_of_requests, outcount, array_of_indices, array_of_statuses);

// MPI_Waitall - Wait for all requests to complete
MPI_Waitall(count, array_of_requests, array_of_statuses);
```

::: callout
**MPI_Request_null**: When a request handle is set to `MPI_REQUEST_NULL`, any wait or test function called with it will return immediately. The status will be empty as there's no operation to report on.
:::

### Non-blocking Testing: The Test Family

These functions check completion status without blocking:

```c
// MPI_Test - Test if a single request has completed
MPI_Test(request, flag, status);

// MPI_Testany - Test if any one of several requests has completed
MPI_Testany(count, array_of_requests, index, flag, status);

// MPI_Testsome - Test if some requests have completed
MPI_Testsome(incount, array_of_requests, outcount, array_of_indices, array_of_statuses);

// MPI_Testall - Test if all requests have completed
MPI_Testall(count, array_of_requests, flag, array_of_statuses);
```

::: callout
**Key Difference**: Test functions return immediately with a boolean flag indicating whether completion has occurred. When `flag` is true, the behavior is equivalent to having called the corresponding wait function.
:::

## Local vs. Remote Completion

::: instructor
Critical concept for students to understand! Emphasize that MPI only guarantees local completion, not remote completion.
:::

::: callout
**Local Completion ≠ Remote Completion**: When MPI reports that a non-blocking send has completed locally, it means:
- The send buffer can be safely reused
- The data has been transferred to the network layer
- **It does NOT guarantee** that the data has been received on the remote side

The send modes provide some indication about reception completion, but MPI does not provide built-in functions to query remote completion. Applications typically don't need this information - they only need to know when they can safely reuse their local buffers.
:::

## Practical Considerations and Performance

### Asynchronous Progress

MPI implementations typically provide **asynchronous progress** - communication continues in the background while your program executes:

::: callout
**Progress Guarantee**: While your program is executing within MPI (e.g., in collective operations, barriers, or other MPI functions), MPI implementations will attempt to make progress on incomplete non-blocking operations. This enables computation-communication overlap.
:::

### Progress Threads: Trade-offs

Some MPI implementations offer **progress threads** - dedicated threads that continuously attempt to progress communication:

::: callout
**Performance Trade-off**: Progress threads consume computational resources that might impact:
- Application thread performance
- Cache behavior
- Overall application performance

These are typically **disabled by default** and should be enabled only after profiling shows benefit for your specific application.
:::

### Hardware Support

Modern network hardware can provide additional support:

::: callout
**RDMA (Remote Direct Memory Access)**: For contiguous buffers, network cards can directly transfer data from memory to memory without CPU intervention. This can enable true computation-communication overlap where your CPU continues executing while the network card handles the data transfer.
:::

## Deadlock Prevention with Non-blocking Communication

::: instructor
Connect back to the deadlock examples from the previous episode. Show how non-blocking operations provide a solution.
:::

One of the most powerful applications of non-blocking communication is **deadlock prevention**. Consider the classic deadlock scenario:

```c
// Deadlock scenario with blocking operations
if (rank == 0) {
    MPI_Send(buffer, count, MPI_INT, 1, tag, comm);
    MPI_Recv(buffer, count, MPI_INT, 1, tag, comm, &status);
} else if (rank == 1) {
    MPI_Send(buffer, count, MPI_INT, 0, tag, comm);
    MPI_Recv(buffer, count, MPI_INT, 0, tag, comm, &status);
}
```

With non-blocking operations, we can break this deadlock:

```c
// Non-blocking solution to prevent deadlock
// Post receives first to establish matching
MPI_Irecv(recv_buffer, count, MPI_INT, (rank+1)%size, tag, comm, &recv_request);

// Then post sends - they can now match with the posted receives
MPI_Isend(send_buffer, count, MPI_INT, (rank+1)%size, tag, comm, &send_request);

// Wait for both operations to complete
MPI_Wait(&recv_request, &status);
MPI_Wait(&send_request, &status);
```

::: callout
**Deadlock Prevention**: By pre-posting receives before sending, we ensure that send operations have matching receives available, eliminating the deadlock condition. The progress within MPI functions ensures these operations can complete.
:::

## Domain Decomposition Example

For applications using domain decomposition (e.g., stencil codes):

```c
// Initiate all communications first
for (int neighbor = 0; neighbor < num_neighbors; neighbor++) {
    MPI_Isend(send_buffers[neighbor], count, MPI_INT,
              neighbors[neighbor], tag, comm, &requests[neighbor]);
}

// Do useful computation here while communication progresses

// Wait for all communications to complete
MPI_Waitall(num_neighbors, requests, MPI_STATUSES_IGNORE);
```

This pattern:
- Enables computation-communication overlap
- Simplifies deadlock avoidance
- Scales better than ordered completion approaches

## Best Practices and Pitfalls

::: critical
- **Do not modify buffers** between initiation and completion of non-blocking operations
- **Prefer simpler completion functions** (wait, test) unless profiling shows benefit of others
- **Enable progress threads cautiously** - measure before and after
- **Use waitall when waiting for all requests** - simpler and often more efficient
- **Remember**: Local completion ≠ remote completion - don't assume remote side has processed data
:::

::: callout
**Premature Optimization Warning**: Functions like `MPI_Waitsome` and `MPI_Testsome` can complicate your code significantly. Only use them if profiling shows they provide measurable benefit. The marginal performance gains rarely justify the added complexity.
:::

## Summary

::: instructor
Recap the key takeaways and connect to future topics.
:::

Today we've covered the fundamental concepts of non-blocking point-to-point communication:

1. **Separation of concerns**: Initiation vs. completion via request handles
2. **Two completion families**: Blocking (wait) vs. non-blocking (test)
3. **Local completion semantics**: What MPI guarantees (and doesn't guarantee)
4. **Practical applications**: Deadlock prevention and computation-communication overlap
5. **Performance considerations**: Progress mechanisms and their costs

These concepts will serve as the foundation for:
- Non-blocking collective operations
- Parallel file I/O
- Advanced MPI programming patterns

::: keypoints
- Non-blocking communication separates operation initiation from completion
- Request handles (MPI_Request) track the status of ongoing operations
- Blocking completion functions (wait, waitany, waitsome, waitall) suspend execution until completion
- Non-blocking test functions (test, testany, testsome, testall) check completion without blocking
- Local completion ≠ remote completion - MPI guarantees local completion but not remote completion
- Non-blocking operations enable computation-communication overlap and deadlock avoidance
- Progress is typically handled within MPI calls unless explicit progress threads are used
:::
