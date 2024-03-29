+++
title = "p1: Printing and allocating"
summary = "Bring back printline-debugging (featuring an overview of memory-mapped I/O and the idea of safely abstracting over unsafe code), plus initialisation of a kernel heap."
date = 2023-03-08T00:00:00-06:00
draft = false
slug = "p1-printing-and-allocating"
toc = true
+++

## 📝 Overview

The goal of this project is to create a safe abstraction for printing characters out to the console and to initialize a kernel heap for dynamic allocation.

If you have not already, read [*p0: Running Rust code on RISC-V in QEMU*]({{< ref "/posts/2023/03/05/rust-operating-system-2" >}}) to set up your development environment and scaffold the project. If you have no idea what any of this is for, check out [*Writing a RISC-V operating system kernel in Rust*]({{< ref "/posts/2023/03/04/rust-operating-system-1" >}}).

This project is still undergoing review and is subject to change.

## 🖨️ Printing to the console

Adding print statements is an extremely useful technique for quickly identifying bugs and letting the world know what is happening inside our kernel.

The CPU typically runs much faster than serial communication protocols, sending bursts of characters at the same time (during one print statement, for example) with long idle periods in between. So hardware typically includes a separate universal asynchronous receiver/transmitter (UART) chip which implements **serial** communication with the outside world. How do we talk to the UART circuitry?

### Introduction to memory-mapped I/O

You could imagine it might be nice for the hardware to have a special instruction, say, `mv-to-console`, which took a register argument and just sent the value of that register out to the console. Alas, then we would also want to add `mv-to-screen` when it comes time to display images, and `mv-to-sound` for sound cards, and so on. We would need to keep adding instructions for each device, which is not very scalable and limits our ability to invent new types of devices that are compatible with older processors.

Instead, maybe we could have an instruction called `iomv` that takes two registers: one specifies the device to send data to, and the other actually contains the data to send. That solves our scalability problem, but we have still increased the complexity of our ISA. (This is how i386 handles some I/O.) More fundamentally, this technique treats I/O addresses as if they existed in a completely separate **address space**. Special functions need to be written to interact with I/O ports and copy data to and from the address space of DRAM.

Modern architectures instead use an abstraction called **memory-mapped I/O** (**mmio**). Whenever a core issues an instruction that interacts with memory (either load or store), there is special circuitry that determines what **device** the target address actually refers to. Each device can claim a range of addresses so it can implement different features depending on what address within its range is used. I/O devices and DRAM *share an address space*, so we can copy data to and from devices using our normal load and store instructions!

The correspondence between addresses and devices, called the **memory map**, is platform-dependent and typically specified in a manual. We've actually already investigated this in p0: DRAM is just another device in the memory map! QEMU so helpfully initialises the memory map of our virtual machine [here](https://github.com/qemu/qemu/blob/master/hw/riscv/virt.c#L77). We're interested in the `VIRT_UART0` device, which apparently starts at `0x10000000` and goes for `0x100` bytes. We'll see later that it is possible to programmatically discover these mappings, if a bit tedious.

Most machines implement the NS16550A UART device, so we can consult some [online resources](https://www.lammertbies.nl/comm/info/serial-uart) to figure out what addresses within the mapped range do interesting things. This is extremely low-level and not very relevant to operating systems, so we'll fast forward and just configure it reasonably:

```rust

extern "C" fn entry() -> ! {
  {
    use core::ptr::write_volatile;
    let addr = 0x1000_0000 as *mut u8;
    // Set data size to 8 bits.
    unsafe { write_volatile(addr.offset(3), 0b11) };
    // Enable FIFO.
    unsafe { write_volatile(addr.offset(2), 0b1) };
    // Enable receiver buffer interrupts.
    unsafe { write_volatile(addr.offset(1), 0b1) };
  }

  // UART is now set up! Let's print a message.
  for byte in "Hello, world!\n".bytes() {
    unsafe { write_volatile(addr, byte) };
  }

  loop {}
}
```

We use `core::ptr::write_volatile` to tell the compiler it *must not* re-order or elide any of our loads/stores. Otherwise, the compiler is allowed to be clever about memory operations (for example, look what happens [here](https://godbolt.org/z/n4hf7TWY6)), which is not what we want when interacting with I/O devices.

We also need to tell QEMU to actually emulate the serial device by adding `-serial mon:stdio` to the `runner` option in `.cargo/config.toml`. Now, running the kernel with `cargo run` should cause our special message to get printed in the QEMU console!

### A safe abstraction for UART

We've been using `unsafe` a lot in our code, and that's not necessarily bad, but we need to be careful to justify why our unsafe code will not lead to undefined behaviour. For code that is often repeated (with the same justification), it makes a lot of sense to refactor the unsafe code out into its own library, and provide safe abstraction around it. The abstraction is then responsible for making sure any special conditions required by the unsafe code are satisfied.

Let's create a new **module** for our UART driver. In `main.rs`, add

```rust
// src/main.rs
// ...

mod uart;
```

Then create a new file called `src/uart.rs`:

```rust
// src/uart.rs

//! This module provides access to the UART console.

/// Represents an initialised UART device.
pub struct Device {
  base: usize
}
```

`uart::Device` will be our safe abstraction: as long as the `base` field is valid, then methods on `Device` will be safe.

```rust
// ...

impl Device {
  /// Create a new UART device.
  /// # Safety
  /// `base` must be the base address of a UART device.
  pub unsafe fn new(base: usize) -> Self {
    use core::ptr::write_volatile;
    let addr = base as *mut u8;
    // Set data size to 8 bits.
    unsafe { write_volatile(addr.offset(3), 0b11) };
    // Enable FIFO.
    unsafe { write_volatile(addr.offset(2), 0b1) };
    // Enable receiver buffer interrupts.
    unsafe { write_volatile(addr.offset(1), 0b1) };
    // Return a new, initialised UART device.
    Device { base }
  }

  pub fn put(&mut self, character: u8) {
    let ptr = self.base as *mut u8;
    // UNSAFE: fine as long as self.base is valid
    unsafe { core::ptr::write_volatile(ptr, character); }
  }
}
```

This might seem dangerous because the caller of `Device::put` has no idea that there might be unsafe things happening inside, but the only way for someone outside of the `uart` module to create a new `Device` is by calling `Device::new`, which explicitly states the requirements that the caller should verify to avoid undefined behavior. So we will still need to use `unsafe` once to obtain an initialised UART device, but not every time we want to write a character.

It is generally a good practice to leave a comment next to `unsafe {}` code blocks explaining what invariants the block might violate and why it should not lead to undefined behaviour. Likewise, unsafe functions should explain what invariants the caller should uphold to avoid undefined behaviour.

Our code for `entry` can be greatly simplified now:

```rust
extern "C" fn entry() -> ! {
  // UNSAFE: correct address for QEMU virt device
  let console = unsafe { uart::Device::new(0x1000_0000) };
  for byte in "Hello, world!".bytes() {
    console.put(byte);
  }
  loop {}
}
```

We'd still like to be able to use `println!` (the Rust macro for printing) with our heap. To do so, we'll need to create a static global variable, called `CONSOLE`, and a macro to write to it.

Add the following to the UART driver:

```rust
// src/uart.rs

// ...

use spinning_top::Spinlock;

static CONSOLE: Spinlock<Option<Device>> = Spinlock::new(None);
```

`Spinlock` is provided by a crate called `spinning_top`. You can add it to your dependencies with `cargo add spinning_top`.

We've declared a static variable called `CONSOLE`. In general, static variables are unsafe to access or modify because the compiler cannot guarantee the absence of data races in the presence of multiple cores. `Spinlock` is a safe abstraction that forces the cores to synchronize their accesses by "spinning" (using an atomic variable) until it is safe to proceed. We use an `Option` becasue the `Device` will start out uninitialised until we call `uart::init` (defined below). Any attempts to print should check first whether the console has been initialised!

Before we can use any global or static variables from Rust, we need to set every byte in the BSS segment to `0`. But where is the BSS? Our linker script already defines two symbols, `_bss_start` and `_bss_end`! We can do this in Rust, but it’s pretty straightforward to implement in assembly in our `_start` function.

<details>
<summary>EXERCISE: write code in assembly to clear the BSS. (Click for answer.)</summary>
    
```rust
// src/main.rs

// ...
    "la sp, _init_stack_top",

    // clear the BSS
    "la t0, _bss_start",
    "la t1, _bss_end",
    "bgeu t0, t1, 2f",
"1:",
    "sb zero, 0(t0)",
    "addi t0, t0, 1",
    "bne t0, t1, 1b",
"2:",
    // BSS is clear!
```

You can verify in GDB that everything between `_bss_start` (inclusive) and `_bss_end` (exclusive) is zero.
</details></p>

Now that we know the BSS segment is cleared, we can write code that uses global and static variables.

``` rust
// src/uart.rs

// ...

/// Initialise the UART debugging console.
/// # Safety
/// `base` must point to the base address of a UART device.
pub unsafe fn init_console(base: usize) {
  let mut console = CONSOLE.lock();
  *console = Some(unsafe { Device::new(base)} );
}

/// Prints a formatted string to the [CONSOLE].
#[macro_export]
macro_rules! print {
    ($($arg:tt)*) => ({
        use core::fmt::Write;
        $crate::uart::CONSOLE.lock().as_mut().map(|writer| {
            writer.write_fmt(format_args!($($arg)*)).unwrap()
        });
    });
}

/// println prints a formatted string to the [CONSOLE] with a trailing newline character.
#[macro_export]
macro_rules! println {
    ($fmt:expr) => ($crate::print!(concat!($fmt, "\n")));
    ($fmt:expr, $($arg:tt)*) => ($crate::print!(concat!($fmt, "\n"), $($arg)*));
}
```

The macro syntax is magic and totally unimportant for now; the only important part is that we need to implement the `core::fmt::Write` trait for `Device` to be able to use the `concat!` macro, which is provided by the built-in `core` crate.

```rust
// src/uart.rs

// ...

impl core::fmt::Write for Device {
  fn write_str(&mut self, s: &str) -> core::fmt::Result {
    for c in s.bytes() {
        self.put(c);
    }
    Ok(()) // there are never errors writing to UART :)
  }
}
```

We can once again simplify the `entry` function!

```rust
// src/main.rs

// ...

extern "C" fn entry() -> ! {
    println!("This should not print because the console is not initialised.");

    unsafe { uart::init_console(0x1000_0000) };

    println!("Hello, world!");

    // ...
}
```

Wrapping unsafe code in a safe abstraction is such a common pattern that it is referenced in [the Rust Book](https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html#unsafe-superpowers) as a best-practice when dealing with unsafe code.

### Reading keyboard input

Another convenient thing UART allows us to do is to accept keyboard input from the console. To do so, we'll add a method, `Device::get`, which checks for any new input and returns one character at a time.

```rust
// src/uart.rs

impl Device {
  // ...

  /// Gets a character from the UART input.
  pub fn get(&mut self) -> Option<u8> {
    const READY: u8 = 0b1;

    let ptr = self.base as *mut u8;
    // SAFETY: Fine as long as base is correct.
    let lsr = unsafe { ptr.offset(5) };
    if core::ptr::read_volatile(lsr) & READY {
      Some(core::ptr::read_volatile(ptr))
    } else {
      None
    }
  }
}
```

Then we'll update our `loop {}` in `entry` to just read characters and print them back out:

```rust
extern "C" fn entry() -> ! {
  // ...

  println!("Hello, world!");

  // print user input back to the console
  loop {
    let c = uart::CONSOLE.lock().as_mut().and_then(uart::Device::get);
    if let Some(c) = c {
      print!("{}", c as char);
    }
  }
}
```

If you run the kernel now, you should be able to type into the QEMU console and see your message appear! This method of dealing with asynchronous logic is called *polling*: the processor repeatedly checks whether new data is available. As you might guess, polling is not very efficient: there might be one new character per dozen milliseconds, but the processor will check hundreds of thousands of times per character. Soon we will explore more efficient approaches to interfacing with I/O devices.

## 📦 Initialising the kernel heap

The kernel, like any complex piece of software, might eventually want to dynamically allocate memory. To do so, we need to set up a heap and make sure Rust knows to use it.

We assume familiarity with dynamic allocation (`malloc`/`free`), so we'll go ahead and use a library for our implementation. The library provides a safe abstraction over some very unsafe raw memory manipulation. Modify the package manifest to add:

```toml
# below [dependencies]

linked_list_allocator = "0.10.5"
```

and run `cargo check` to update the dependency. Next, add a new module for the heap:

```rust
// src/main.rs

// ...
mod heap;
```

and in a new file:

```rust
// src/heap.rs
//! Provides the kernel heap.

use linked_list_allocator::LockedHeap;

#[global_allocator]
static ALLOCATOR: LockedHeap = LockedHeap::empty();
```

The `#[global_allocator]` attribute tells Rust that this is the heap it should use for anything that requires dynamic allocation. Having a global allocator isn't strictly necessary, but it gives us access to the built-in `alloc` crate, which provides many useful data structures.

Before we can use the heap, we need to initialise it. But how do we know what region of memory is safe to use? Let the linker tell us!

```
# src/script.ld

SECTIONS {
  # ...
  PROVIDE(_kernel_heap_bottom = _init_stack_top);
  PROVIDE(_kernel_heap_top = ORIGIN(ram) + LENGTH(ram));
  PROVIDE(_kernel_heap_size = _kernel_heap_top - _kernel_heap_bottom);
}
```

This will create new symbols which refer to the entire region of physical memory after the stack. (We might want to shrink the kernel heap later, but for now this is perfectly fine.)

In `heap.rs`, we'll declare an initialisation routine:

```rust
// src/heap.rs

/// Initialise the kernel heap.
/// # Safety
/// Must be called at most once.
pub unsafe fn init() {
    let heap_bottom;
    let heap_size;
    // UNSAFE: This is fine, just loading some constants.
    unsafe {
        // using inline assembly is easier to access linker constants
        asm!(
          "la {heap_bottom}, _kernel_heap_bottom",
          "la {heap_size}, _kernel_heap_size",
          heap_bottom = out(reg) heap_bottom,
          heap_size = out(reg) heap_size,
          options(nomem)
        )
    };
    println!(
        "Initialising kernel heap (bottom: {:#x}, size: {:#x})",
        heap_bottom as usize, heap_size
    );
    // UNSAFE: Fine to call at most once.
    unsafe { ALLOCATOR.lock().init(heap_bottom, heap_size) };
}
```

You should take a look at the [documentation](https://docs.rs/linked_list_allocator/0.10.5/linked_list_allocator/struct.Heap.html#method.init) for `linked_list_allocator::Heap::init` to see why our initialisation call does not cause undefined behavior.

Now we just need to call `heap::init` from `entry`:

```rust
// src/main.rs

// ...

extern "C" fn entry() -> ! {
  // ...
  
  // UNSAFE: Called exactly once, right here.
  unsafe { heap::init() };

  // Now we're free to use dynamic allocation!
  {
    extern crate alloc;
    use alloc::boxed::Box;
    let my_heap_pointer = Box::new(10);
    // my_heap_pointer lives on the heap!
  }

  // ...
}
```

That's all for this post. Next time we will set up a testing harness to incorporate unit tests into the kernel.