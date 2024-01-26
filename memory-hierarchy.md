# Memory Hierarchy Design



## Memory Technology and Optimizations

### Teminology: 
- **Access Time**: Time between when a read is requested and when the desired word arrives.
- **Cycle Time**: Minimum time between unrelated requests to memory
- **DRAM**: Dynamic RAM
- **SRAM**: Static DRAM </br>
_Note: Since 1975, virtually all computers have used DRAM for main memory and SRAM for caches._
- **SDRAM**: Synchronous DRAM (Added clock)
- **Hard errors/permanent faults**: occur during fabricatino, as well as from a circuit change during operation (e.g., failure of a Flash memory cell after many writes). 
- **Soft errors/transient faults**: also known as _dynamic errors_, are changes to a cell's contents, not a change in the circuitry. They can be detected by parity bits and detected and fixed by error correcting codes (ECC).
- **Write through**: Cache with write through, data is simultaneously updated to cache and memory. This process is simpler and more reliable.
- **Write back**: The data is updated only in the cache and updated into the memory at a later time. Data is updated in the memory only when the block is ready to be replaced. Each block in the cache needs a bit to indicate  if the data present in the cache was modified (dirty) or not modified (clean). If the block being replaced is clean, there's no need to write it back to memory since they hold the same data.


### DRAM Technology
The dynamic nature of the circuits in DRAM required data to be written back after being read. Thus the differnece between the **access time** and the **cycle time** as well as the need to refresh (every, i.e. 64ms the controller reads and writes back the memory to avoid cells leaking charge thus losing data).
DRAMs use 1 transistor per bit (acting as a capacitor).
This means:
- The sesning wires that detect the charge must be prechardged, which sets them "halfway" between a logical 0 and 1, allowing the small charge stored in the cell to cause a 0 or 1.
- On reading, a row is palced into a row buffer, where CAS signals can select a portion of the row to read out from the DRAM.
- Because reading a row destroys the information, it must be written back when when the row is no longer needed.This write back happens in overlapped fashion.
- In addition, to prevent loss of information as the charge in a cell leaks away (assuming it is not read or written), each bit must be “refreshed” periodically

DRAMs multiplex the address lines, therby cutting the address pin numbers in half. One-half of the address is sent first during the _row access strobe_ (RAS). The other half of the address, sent during the _column access strobe_ (CAS), follows it. Memory is organized as a rectangular matrix addressed by rows and columns.


### SRAM Technology
SRAMs don't need to refresh, so their **access time** is very close to the **cycle time**.
SRAMs use 6 transistors per bit to prevent the information from being disturbed when read (no write back needed).
Cache SRAMs are normally organized with a width that matches the block size of the cache, with the tags stored in parallel to each block. This allows an entire block to be read out or written into a single cycle. This capability is partic- ularly useful when writing data fetched after a miss into the cache or when writing back a block that must be evicted from the cache.

The access time to the cache (ignoring the hit detection and selection in a set associative cache) is propportional to the number of blocks in teh cache, whereas the energy consumptoin depends both on the number of bits in the cache (static power) and on the number of blocks (dynamic power).


### Flash Memory
Flash memory is a type of EEPROM (electronically erasable programmable read-only memory), which is normally read-only but can be erased. The other key property of Flash memory is that it holds its contents without any power. We focus on NAND Flash, which has higher density than NOR Flash and is more suitable for large scale.

Differences with standard DRAM:
- Reads to Flash are sequential and read an entire page (512 bytes, 2 KiB or 4KiB). This creates a long delay to access the first byte from a random address (about 25 μS), but can supply the remainder of the page block at about 40 MiB/s.
(DDR4 SDRAM takes 40 ns to first byte and can transfer the rest at 4.8 GiB/s, however, Flash is 300 to 500 faster than magnetic disks)
- Flash memory must be erased (in blocks, not bytes nor words) before it is overwritten. For writing Flash is about 1500 times slower than SDRAM, and about 8-15 times faster than magnetic disks. 
- Flash memory is nonvolatile and draws significantly less power when not reading or writing.
- Flash memory limits the number of times that any given block can be writen, typically at least 100,000. Flash memory controllers handle the technique to uniformly distribute blcok writes, also called _write leveling_.
- Currently cheaper than SDRAM but more expensive than disks.


## Virtual Memory

Virutal memory addresses do **NOT** have the same structure as physical addresses. Virutal memory addresses are split in 2, with the upper bits being the Virtual Page Number (from the Page Table) and the remaining bits being the Page offset (to the requested byte/word in the page). 

Good illustration of the aliasing problem of VIVT (Virtually Indexed Virtually Tagged) caches: https://www.youtube.com/watch?v=Tg6ID2uWjuY


