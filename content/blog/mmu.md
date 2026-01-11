+++
title = "Memory Management"
date = 2026-01-11
+++

I've been building my own [OS](https://github.com/ravi-sanker/rasOS) and I'm struggling to keep the mental models for understanding memory management straight in my head. I'm writing this post to revisit the concepts, both from a theoretical POV and a practical POV through the x86 architecture. 

References:
- [OSTEP](https://pages.cs.wisc.edu/~remzi/OSTEP/)
- [osdev](https://wiki.osdev.org/)


---

# Theory

## Why?

Memory is just an array of bytes addressable through the address bus. If your address bus has 20 lines, you have 1MiB worth of addressable memory. Why do we need to complicate this further? If you want to write or read from a specific place in memory, why can't we just provide the address and call it a day?

The problem arises if your OS wants to support user programs and let's face it, that's one of the most fundamental requirements of an OS. If you're a user program, you shouldn't have the ability to read or write to memory belonging to other programs. Well in that case, can't my user program have access to all of memory (except the kernel code) and when it's time to switch programs, swap the current user program's memory to disk and then load the next program's memory from disk? This is horrifyingly slow. Another problem is that we don't want the user program to worry about where in memory it needs to store its data, code etc. 

So we need some sophisticated methods to provide each user program the illusion that it has access to the entire address space (for ease of use), but also make this address translation from the virtual address to the real physical address efficient. We also need to make sure the user program cannot access memory that doesn't belong to it.

## Base and Bounds

The simplest way to achieve the above requirements is to introduce two new registers in the hardware - base and bounds. To get your physical address, simply add it to the value in the base register. To achieve protection, make sure it doesn't cross the value in the bounds register.

The problem with this is that a program typically contains three logical "segments" - code, stack and heap. Code sits at the top (lower address) followed by the heap. The stack is at the bottom (higher address). Heap grows positively while the stack grows negatively. There's a lot of empty space in between that gets wasted. This is called **internal fragmentation**. 

## Segmentation

The solution to internal fragmentation is to have these base and bounds registers for each logical segment. From the address, how would you know which segment to use? We could assign the starting 'n' bits to be selector for the segment. But this leads to another problem - **external fragmentation**. You have these tiny slices of used memory spread all over the memory that it becomes a nightmare to find a long continuous block of memory.

## Paging

Though many solutions have been proposed for external fragmentation, maybe it's time to solve this problem a different way. Instead of dividing the memory into dynamic blocks, let's divide into blocks of a fixed size. Let's call each such block a page. Ok then? Let's have a page table that stores a mapping between the virtual page number to the physical page (called frame) number. This page table will be unique for each process. In your address, the top few bits will refer to which page and the remaining will be offset within that page. You can also have permission information stored in this table. The problem with this is the page table will be too large. So instead you will have multi-level page tables. 

## Efficiency

For efficiency, the only way is to make the hardware take care of the address translation. Note that it'll still be job of the OS to make sure the appropriate registers and page tables are set when swapping processes. But the actual address translation from virtual address to physical address should be completely taken care of by the hardware.

In the case of paging, the hardware can support something called as the TLB (Translation Lookaside Buffer). This basically acts as a cache for the page number to physical frame number mapping.


---

# Practical (x86)

So we looked at the theory, but the intricate details of the practical implementation are quite hard to wrap your head around. 

## Real Mode

This is the default mode in which your hardware is initialized. This exists only for backward compatibility reasons. There is no support for virtualization of memory. You have 1MiB of addressable space, which means you can only use 20 address lines. You specify the address with a combination of segment registers and "pointer" registers. Both are 16 bits. To get a 20 bits address, you left shift the segment register value by 4 and add the "pointer" register value. 

## Protected Mode

This is where the fun starts. You can implement both segmentation and paging. But segmentation is very outdated and no one uses it anymore.

### Segmentation in x86

Unfortunately, it's not as simple as what was specified in the theory with using a pair of base and bounds register for each logical segment. Reality is different.

x86 has a concept called "descriptor tables". There are two descriptor tables - global and local. Let's take a 16 bit segment register as an example - bits 15-3 will tell you which entry in the descriptor table to use, the next bit (bit 2) will tell you whether to use the local or the global one. Bits 1-0 are related to permissions. So you go to the appropriate descriptor table entry and in that table entry, the actual physical address of the starting of the segment is stored (in a weird format, but I don't want to get into the specifics). The limits for the segment is also stored. So basically your segment registers contain the offset into one of the two descriptor tables.

How do you where the descriptor tables are stored? Through registers - GDTR holds GDT location. LDTR holds a selector pointing to LDT descriptor in GDT. There are many nuances here that I'm skipping as this is stupidly complicated. 

So if you have a virtual address, how do you know which segment register to use? The virtual address here isn't like how you envision in theory. You are basically stuck with assembly. Refer to "Notes Regarding C" in https://wiki.osdev.org/Segmentation.

### Paging in x86

This is quite similar to what was discussed in theory. Let's take 32 bit addressable space as an example. The page size is 4KiB. So you need 12 bits to know the offset within the page. You have 20 bits remaining. You could choose the entire 20 bits to store the page number, but instead intel has chosen a multi-level page table structure. The first 10 bits (31-22) will tell you the offset into the first table (called the page directory). This entry will give you the address of the page table. The next 10 bits (21-12) will give you the offset into this page table. The entry in the page table will give you the starting address of your page in physical memory. 

So 32 bits is split into 10 + 10 + 12. There are 1024 page directory entries and 1024 page table entries. The page size is 4KiB.

How do you know where the page directory table is stored? This is stored in a register called CR3. You also need to enable the paging bit in the CR0 register. Note that while swapping processes, you can just update the CR3 register and your new process will have a completely different virtual to physical address mapping as it's pointing to a completely different page table.
