
# Multiport Serial Driver Analysis

## Overview
The SerialWDM driver supports **Multiport Serial Cards** (multiple UARTs on a single card sharing an interrupt) and **Shared Interrupts** (multiple cards sharing an IRQ line). The implementation uses a dynamic ISR dispatch mechanism to route interrupts to the correct port instance.

## 1. Key Data Structures
**File**: `serial.h`

The `SERIAL_DEVICE_EXTENSION` structure contains specific fields to link related ports together:

-   `TopLevelOurIsr`: The actual ISR routine to call (e.g., `SerialISR`).
-   `TopLevelOurIsrContext`: The context for the above ISR (usually the extension itself).
-   `TopLevelSharers`: A doubly-linked list (`LIST_ENTRY`) connecting all devices sharing the same **System Interrupt Vector**.
-   `MultiportSiblings`: A doubly-linked list connecting all ports residing on the **same physical multiport card**.
-   `PortOnAMultiportCard`: Boolean flag.
-   `PortIndex`: The 1-based index of the port on the card (1, 2, 3...).

The `SERIAL_MULTIPORT_DISPATCH` structure is used when a card has a single "Interrupt Status Register" that indicates which port is interrupting:

```c
typedef struct _SERIAL_MULTIPORT_DISPATCH {
    PSERIAL_DEVICE_EXTENSION Extensions[SERIAL_MAX_PORTS_INDEXED]; // e.g. 16 ports
    PUCHAR InterruptStatus; // Hardware register address
    ULONG UsablePortMask;   // Bitmask of active ports
    // ...
} SERIAL_MULTIPORT_DISPATCH;
```

## 2. Initialization & Conversion Logic
**File**: `initunlo.c`

The driver does not assume a multiport configuration upfront. Instead, it **dynamically promotes** a single port to a multiport configuration when it detects a conflict or a shared resource during initialization.

### `SerialSingleToMulti`
When a new device is started that shares an interrupt with an existing device (and is identified as part of the same multiport card via registry), the driver:
1.  Allocates a `SERIAL_MULTIPORT_DISPATCH` structure.
2.  Populates it with the existing device's extension.
3.  **Replaces the connected ISR**: The system interrupt vector is updated to point to a "Dispatcher ISR" (like `SerialIndexedMultiportIsr`) instead of the standard `SerialISR`.
4.  The original `SerialISR` is demoted to be called *by* the dispatcher.

### `SerialCleanLists`
When a device is removed:
1.  It removes itself from `TopLevelSharers` and `MultiportSiblings` lists.
2.  If it was the "root" of a shared interrupt, ownership is transferred to the next sibling in the list to ensure the interrupt is still serviced.

## 3. ISR Strategy for Multiport
**File**: `isr.c`

The driver provides specialized ISRs for different hardware topologies:

1.  **`SerialSharerIsr`**: Used for generic shared interrupts (e.g., two PCI cards on IRQ 5). It walks the `TopLevelSharers` linked list and calls the ISR for every device until one claims the interrupt.
2.  **`SerialIndexedMultiportIsr`**: Used for cards where an 8-bit register contains the **index number** (0-15) of the interrupting port.
3.  **`SerialBitMappedMultiportIsr`**: Used for cards where an 8-bit/16-bit register has a **bitmask** (bit 0 = port 1, bit 1 = port 2) indicating active interrupts.

### Logic Flow
```text
[Hardware Interrupt]
      |
      v
[System IDT -> SerialSharerIsr or SerialIndexedMultiportIsr]
      |
      +-> [Check Interrupt/Status Register]
      |
      +-> [Identify Target Port(s)]
            |
            v
      [Call Extension->TopLevelOurIsr (Standard SerialISR)]
            |
            v
      [Handle UART (IIR, RBR, THR)]
```

## 4. Registry Configuration
**File**: `registry.c`

The behavior is driven by registry values read during `DriverEntry` or `SerialAddDevice`. Key values include:
-   `PortIndex`: Logical number of the port.
-   `MultiportDevice`: Indicates if the device is part of a multiport card.
-   `MaskInverted`: HW specific quirck (active low interrupt bits).
-   `PermitShare`: Authorization to enable the sharing logic.

## Summary
The implementation is highly modular. It treats "Shared Interrupts" and "Multiport Cards" as variations of the same problem: mapping one System IRQ to multiple Device Contexts. The use of linked lists allows for dynamic addition/removal of ports without recompiling the driver.
