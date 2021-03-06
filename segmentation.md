# Segmentation

TODO this section is a mess. Organize and slit it up.

Described on the AMD64 manual vol. 2 Chapter 4 - Segmentation.

Makes the transition from logical to linear addresses. Linear addresses are then converted to physical addresses (those that go to RAM wires) by the paging circuits:

    (logical) ------------------> (linear) ------------> (physical)
                 segmentation                 paging

Segmentation does not exist anymore in x86-64 64-bit mode: only in compatibility mode.

This feature was initially meant to be used to implement process virtual memory spaces, but this usage has been mostly supplanted by a later implemented feature called paging. This is probably why it was dropped in x86-64.

The main difference between paging and segmentation is that the size of pages if fixed, while the size of segments can vary and is indicated inside the segment descriptor.

Besides address translation, the segmentation system also manages of other features, such as the privilege level of execution (ring) functionality which is still widely used, and compatibility mode.

In Linux 32-bit for example, only two segments are used at all times: one at ring 0 for the kernel, and one another at privilege 3 for all user processes. TODO draw.

## Hardware implementation

Like paging, the segmentation transformation needs to happen on every memory read and write. For this reason, it is implemented *by the hardware*.

The processor manual says something like:

- when segmentation is turned on
- I will look for segmentation information (local and global descriptor tables) at the RAM physical address at my register X, which can be read and set by the `gdtr`, `ldtr`, `lgdt` and `lldt` instructions.
- the local and global descriptor tables must have to have the following format
- using that information on RAM, I will decide how to do the address translation and every other function of the segmentation system

It is then OS to set up and manage those RAM data structures and CPU registers to make the CPU do what it wants.

## Global descriptor table

RAM data structure that holds segment information.

The segment information data structure is called a *segment descriptor*.

Each segment descriptor is identified and retrieved via a *segment selector* structure.

### Local descriptor table

TODO vs global?

## Segment selector

A segment selector is a 16 bit structure that identifies the current segment descriptor and the current privilege level.

It has the following fields:

-   index: 13 bits to identify the Segment descriptor within the current table.

    There can therefore be up to 2^13 segment descriptors on a table.

    The current table is determined by the values of `gdtr` and `ldtr` registers and by the TI bit of the segment selector.

-   `RPL`: Request privilege level.

    The privilege level of the code that will execute a Code segment.

-   `TI`: 1 bit table indicator. If set, indicates that this is a local descriptor table.

    Otherwise, it is a global descriptor table.

## Segment descriptor

Segment descriptors are kept in RAM, and are used by the hardware to translate logic to linear addresses.

It is up to the OS to set up and maintain segment descriptors on the RAM, and to inform the hardware of its location via the `gdtr` and `ldtr` registers The OS can modify those registers via the `lgdt` and `lldt` instructions.

Segment descriptors are kept inside tables which contain many contiguous segment descriptors called either global descriptor table or local descriptor table.

Each segment descriptor is 8 bytes long, and contains information such as the following.

-   BASE: 32 bit start address and end address of the segment

-   LIMIT: 20 bit segment length. This is multiplied by $2^12$ if G is set so the maximum length is 4GB ($2^32$).

    Minimum length is 4 Kb.

-   G: granularity flat. If set, LIMIT is in multiples of $2^12$ bytes, else multiples of 1 byte.

-   DPL: 2 bit privilege level.

    Compared to the privilege level of the Segment Selector to determine if users have or not permission to take certain actions ( the rings are based on this )

-   Type: 4 bit type field. Some of the types are:

    - Code: indicates a code segment. It is on this case the permissions to take actions are checked.

    - Data:

    - TSSD: task state segment descriptor. The segment contains saved register values (between process sleeps)

    - LDTD: the segment contains a local descriptor table

-   S: system

    If set, indicates that the RAM of that segment contains important structures such as Local descriptor table.

The current segment descriptor is determined by the current segment selector and the values of the `gdtr` and `ldtr` registers.

## Segment registers

Segment registers contain segment selectors

There are 6 segment registers.

3 have special meanings:

- CS: code segment
- SS: TODO
- DS: data segment

And the other three don't and are free for programmer use.

- ES
- FG
- GS

Segment selectors can be put into those segment registers via `mov` instructions.

Each segment selector has an associated read only register which contains the corresponding segment descriptor to that selector.

Segment descriptors are pulled into dedicated processor registers automatically when a segment register changes value.

This allows to read segment descriptors from RAM only once when segments change, and access them directly from the CPU the following times.

TODO which of those segments are used at each time?

## Segment descriptor types

TODO what is the difference between types?

## Example of address translation

TODO very important. One example, two programs running. Logical to linear address translation.

## Linux

TODO How Linux uses segments.
## GDT

Table in memory that gives properties of segment registers.

Segment registers in protected mode point to entries of that table.

GDT is used as soon as we enter protected mode, so that's why we have to deal with it, but the preferred way of managing program memory spaces is paging.

Format straight from the Linux kernel 4.2: `arch/x86/include/asm/desc_defs.h` in `struct desc_struct`:

    u16 limit0;
    u16 base0;
    unsigned base1: 8, type: 4, s: 1, dpl: 2, p: 1;
    unsigned limit: 4, avl: 1, l: 1, d: 1, g: 1, base2: 8;

- `g`: granularity of the limit. If `0`, 1 byte, if `1`, 4KiB.

Other sources:

- Intel Manual 325384-053US Volume 3, 3.4.5 Segment Descriptors
- https://en.wikipedia.org/wiki/Global_Descriptor_Table
- http://wiki.osdev.org/GDT

### Null segment selector

### Null descriptor

Intel manual 3.4.2 Segment Selectors says:

> The first entry of the GDT is not used by the processor. A segment selector that points to this entry of the GDT (that
is, a segment selector with an index of 0 and the TI flag set to 0) is used as a “null segment selector.” The processor
does not generate an exception when a segment register (other than the CS or SS registers) is loaded with a null
selector. It does, however, generate an exception when a segment register holding a null selector is used to access
memory. A null selector can be used to initialize unused segment registers. Loading the CS or SS register with a null
segment selector causes a general-protection exception (#GP) to be generated.

I think this means that it is impossible to use the first entry. So you can do whatever you want with it?

### Effect on memory access

The GDT modifies every memory access of a given segment by:

- adding an offset to it
- limiting how big the segment is

If an access is made at an offset larger than allowed: TODO some exception happens, which is like an interrupt, and gets handled by a previously registered handler.

The GDT could be used to implement virtual memory by using one segment per program:

    +-----------+--------+--------------------------+
    | Program 1 | Unused | Program 2                |
    +-----------+--------+--------------------------+
    ^           ^        ^                          ^
    |           |        |                          |
    Start1      End1     Start2                     End2

The problem with that is that each program must have one segment, so if we have too many programs, fragmentation will be very large.

Paging gets around this by allowing discontinuous memory ranges of fixed size for each program.

The format of the GDT is given at: http://wiki.osdev.org/Global_Descriptor_Table

### Effect on permissions

Besides fixing segment sizes, the GDT also specifies permissions to the program that is running:

-   ring level: limits several things that can or not be done, in particular:
    - instructions: e.g. no in / out in ring 3
    - register access: e.g. cannot modify control registers like the GDTR in ring 3. Otherwise user programs could just escape restrictions by changing that!
-   executable, readable and writable bits: which operations can be done

## GDTR

## GDT register

In 32-bit, a 6 byte register that holds:

- 2 byte length of the GDT (TODO in bytes or number of entries?)
- 4 byte address of the GDT in memory

In 64 bit, makes 10 bytes, with the address having 8 bytes

GRUB seems to setup one for you: http://www.jamesmolloy.co.uk/tutorial_html/4.-The%20GDT%20and%20IDT.html

## lgdt

Loads the segment description register from memory.

TODO where is it on the Linux kernel?

Candidates:

- linux/arch/x86/kernel/head_64.S
- linux/arch/x86/boot/compressed/head_64.S
