# STM32 FreeRTOS Counting Semaphores and Resource Contention

This repository contains an embedded C project developed for the STM32 Nucleo-F446RE development board. The project demonstrates how to model access control for a limited pool of identical shared resources using a FreeRTOS Counting Semaphore via the CMSIS-RTOS v2 API.

---

## Project Overview

The objective of this project is to manage multi-task resource allocation when the number of identical available resources is greater than one but limited. It also highlights the engineering differences between managing allocation quantity versus protecting data stream atomicity.

* **Kernel Framework:** FreeRTOS managed via the CMSIS-RTOS V2 wrapper layer.
* **Synchronization Primitive:** Counting Semaphore.
  * **Maximum Count:** 2 (representing a pool of two identical shared resources).
  * **Initial Count:** 2 (both resources available at initialization).
* **Core Architecture:**
  * **Tasks A, B, and C:** Three independent tasks with equal priority levels (`osPriorityNormal`) that continuously compete to acquire a token from the shared resource pool.
  * **Resource Simulation:** When a task successfully acquires a token, it simulates workload processing and prints status logs to a single shared communication interface (such as a serial port or SWV ITM console).

---

## Technical Theory

### Counting Semaphores vs. Mutexes
While a Mutex provides strict mutual exclusion (allowing only one thread to own a singleton resource at a time), a Counting Semaphore acts as a token bucket for multiple identical resources. 

* **Parallelism and Throughput:** With a resource count of 2, FreeRTOS allows two tasks (e.g., Task A and Task B) to execute their critical sections simultaneously, maximizing CPU efficiency.
* **Resource Contention:** When Task C attempts to acquire a token while the count is 0, the kernel instantly moves Task C into the **Blocked** state. This relinquishes CPU runtime to prevent busy-waiting until either Task A or Task B releases a token.
* **The Interleaving Risk:** Because a counting semaphore tracks resource *quantity* rather than data *ownership*, multiple tasks executing concurrently might write to a single underlying hardware interface (like a shared UART transmitter) at the exact same time. This can cause character interleaving or garbled data logs, demonstrating why a Mutex is still required if the sub-resource itself demands strict atomicity.

---

## Hardware Requirements

* Development Board: STM32 Nucleo-F446RE (ARM Cortex-M4)
* Connectivity: USB Type-A to Mini-B cable (supporting SWD flash programming and SWV data tracing)

---

## Hardware & Peripheral Configuration

The system initialization profile is established within STM32CubeMX using the following parameters:

| Component / Interface | MCU Pin | Configuration Mode | Function |
| :--- | :--- | :--- | :--- |
| **SYS Debug Mode** | N/A | Trace Asynchronous Sw | Enables SWV ITM Console Output |
| **SYS Timebase Source** | N/A | Hardware Timer (`TIM6`) | Isolated HAL Tick Source |

### Critical System Core Adjustments
* **System Clock:** Configured via the internal oscillator (RCC Bypass mode) running at a stable frequency of **84 MHz**.
* **Timebase Source Selection:** The HAL timebase source is shifted from `SysTick` to hardware timer **TIM6**. FreeRTOS claims exclusive rights to `SysTick` for its 1 ms kernel slicing clock, making this alteration necessary to prevent kernel resource crashes.

---

## FreeRTOS Configuration Attributes

The counting semaphore and concurrent tasks are defined inside the CMSIS-RTOS v2 middleware configuration layer:

### Counting Semaphore Specification
* **Semaphore Name:** `myCountingSemHandle`
* **Maximum Count:** 2
* **Initial Count:** 2

### Task Specifications

| Task Name | Function Entry | Priority Level | Stack Size (Words) |
| :--- | :--- | :--- | :--- |
| **TaskA** | `func_TaskA` | `osPriorityNormal` | 128 |
| **TaskB** | `func_TaskB` | `osPriorityNormal` | 128 |
| **TaskC** | `func_TaskC` | `osPriorityNormal` | 128 |

---

## Core Code Implementation

Each of the three tasks shares a uniform execution pattern inside their respective entry functions:

```c
void func_TaskA(void *argument)
{
  for(;;)
  {
    // Block indefinitely (osWaitForever) if both resource tokens are occupied
    if(osSemaphoreAcquire(myCountingSemHandle, osWaitForever) == osOK)
    {
      // Crucial Section: Task has successfully acquired one of the two resources
      printf("Task A successfully accessed the shared resource pool.\n");
      
      // Simulate task processing workload
      osDelay(500); 
      
      // Release the token back to the semaphore pool
      osSemaphoreRelease(myCountingSemHandle);
      
      // Delay before attempting to re-acquire the resource
      osDelay(100); 
    }
  }
}
