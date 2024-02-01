# Instructino Level Parallelism

## What is Instruction Level Parallelism
All the techniques in this chapter exploit parallelism among instructions. The amount of parallelism available within a basic block—a straight-line code sequence with no branches in except to the entry and no branches out except at the exit—is quite small. To obtain substantial performance enhancements, we must exploit ILP across multiple basic blocks.


```
for (i=0; i<=999; i=i+1)
  x[i] = x[i] + y[i];
```
In this example, each iteration of the loop can overlap with any other iteration, although within each loop iteration, there is little or no opportunity for overlap.


Determining how one instruction depends on another is critical to determining how much parallelism exists in a program and how that parallelism can be exploited. To exploit ILP we must determine which instructions can be executed in parallel.
If two instructions are parallel, they can execute simultaneously in a pipeline of arbitrary depth without causing any stalls. (Assuming the pipeline has sufficient resources)
If two instructions are dependent, they are not parallel and must be executed in order, although they may often be partially overlapped.


Dependences are properties of _programs_. Wheter a given dependence results in an actual hazard being detected and whether that hazard actually causes a stall are properties of the _pipeline organization_.
### Dependences

There are three different types of dependences:
1. _Data dependences_:
  An instruction _j_ is data-dependent on instruction _i_ if either of the following holds:
  - Instruction _i_ produces a result that may be used by instruction _j_.
  - Instruction _j_ is data-dependent on instructino _k_, and instruction _k_ is data-dependent on instruction _i_.

2. _Name dependences_:
  A name dependence occurs when two instructions use the same register or memory location, called a _name_, but there is no flow of data between the instructions associated with that name. There are two types of name dependences between an instruction _i_ that _precedes_ instruction _j_ in program order:
  - An _antidependence_ between instruction _i_ and instruction _j_ occurs when instruction _j_ writes a register or memory location that instruction _i_ reads.
  - An _output dependence_ occurs when instruction _i_ and instruction _j_ write the same register or memory location.
   
3. _Control dependences_:
  In general, two contraints are imposed by control dependences:
  - An instruction that is control-dependent on a branch cannot be moved _before_ teh branch so taht its execution _is no longer controlled_ by the branch. 
  - An instruction that is not control-dependent on a branch cannot be moved _after_ the branch so that its execution _is controlled_ by the branch. For example, we cannot take a statement before the if statement and move it into the then portion.

  When processors preserve strict program order, they ensure that control dependences are also preserved. We may be willing to execute instructions that should not have been executed, however, thereby violating the control dependences, _if_ we can do so without affecting the correctness of the program. Thus, control dependence is not the critical property that must be preserved. Instead, the two properties critical to program correctness (and normally preserved by maintaining both data and control dependences) are the _exception behaviour_ and the _data flow_.


### Hazards
A hazard exists whenever there is a **name** or **data** dependence between instructions, and they are close enough that the overlap during execution would change the order of access to the operand invovled in the dependence. Because of the dependence we must preserve what is called _program order_. The goal of both our software and hardware techniques si to exploit parallelism by preserving program order _only where it affects the outcome of the program_.

### Data hazards
- RAW _(read after write)_ - _j_ tries to read a source before _i_ writes it, so _j_ incorrectly gets the _old_ value. This hazard is the most common type and corresponds to a true data dependence.
- WAW _(write after wrie)_ -  _j_ tries to write an operand before it is written by _i_. The writes end up being performed in teh wrong order, leaving the value writen by _i_ rather than the value written by _j_ in the destination.
- WAR _(write after read)_ -  _j_ tires to write a destionation before it is read by _i_ so _i_ incorrectly gets the _new_ value. This hazard arises from antidependence (or name dependence).

### Basic Pipeline Scheduling and Loop Unrolling
To keep a pipeline full, parallelism among instructions must be exploited by finding sequences of unrelated instructions that can be overlapped in the pipeline.
**To avoid a pipeline stall, the execution of a dependent instruction must be separated from the source instruction by a distance in clock cycles equal to the pipeline latency of that source instruction.**

A simple scheme for increasing the number of instructions relative to the branch and overhead instructions is _loop unrolling_. Unrolling simply replicates the loop body multiple times, adjusting the loop termination code.

## Dynamic Scheduling
In a dynamically scheduled pipeline, all instructions pass through the issue stage in order; however they can be stalled or bypass each other in the second tage and thus enter execution out of order. Instructions will also finish out-of-order.

The entire book keeping is done in hardware. The hardware considers a set of instructions called the instruction window and tries to reschedule the execution of these instructions according to the availability of operands. The hardware maintains the status of each instruction and decides when each of the instructions iwill move from one stage to another. The dynamic scheduler introduces register renaming in hardware and eliminates WAW and WAR hazards.

### Tomasulo's Algorithm
With Tomasulo's algorithm, the register renaming is provided by _reservation stations_ (RSs). Associated with every functional unit, we have a few reservation stations. When an instruction is issued, a reservation station is allocated to it. The resrevation station stores information about the instruction and buffers the operan values (when available). So, the resrevation station fetches and buffers an operand as soon as it becomes available (not necessairly involving register file). This helps in avoiding WAR hazards. If an operand is not available, it stores information about the instruction that supplies the operand. THe renaiming is done through the mapping between the registers and the reservation stations. WHen a functional unit finishes its operation, the result is broadcast on a result bus, called the _common data bus_ (CDB). This value is written to the appropriate register and also the reservation station waiting for that data. When two instructions are to modify the same register, only the last ouput updates the register file, thus handlign WAW hazards. Thus, the register specifiers are renamed with the reservation stations, which may be more than the registers. For load and store operations, we use load and store buffers, which contain data and addresses, and act like reservation stations.

The three steps in a dynamic shecudler are:
1. _Issue_:
  - Get next instruction from FIFO queue
t  - If available RS, issue the instruction to teh RS with operand values if available
  - If a RS is not available, it becomes a structural hazard and the instruction stalls
  - If an earlier instruction is not issued, then susbsequent instructions cannot be issued
  - If operand values are not available, the instructions wait for the operands to arrive on the CDBs

2. _Execute_:
  - When operand becomes available, store it in any reservation station waiting for it
  - When all operands are ready, issue the instruction for execution
  - Loads and store are maintained in program order through teh effective address
  - No instruction allowed to initiate execution until all branches that proceed it in program order have completed

3. _Write result_:
  - Write result on CDB into reservation stations and store buffers
  - Stores must wait until address and value are received


### Hardware Based Speculation
Hardware-based speculation combines three key ideas: (1) dynamic branch prediction to choose which instructions to execute, (2) speculation to allow the execution of instructions before the control dependences are resolved (with the ability to undo the effects of an incorrectly speculated sequence), and (3) dynamic scheduling to deal with the scheduling of different combinations of basic blocks. (In comparison, dynamic scheduling without speculation only partially overlaps basic blocks because it requires that a branch be resolved before actually executing any instructions in the successor basic block.)

To extend Tomasulo’s algorithm to support speculation, we must separate the bypassing of results among instructions, which is needed to execute an instruction speculatively, from the actual completion of an instruction. By making this separation, we can allow an instruction to execute and to bypass its results to other instructions, without allowing the instruction to perform any updates that cannot be undone, until we know that the instruction is no longer speculative.

Using the bypassed value is like performing a speculative register read because we do not know whether the instruction providing the source register value is providing the correct result until the instruction is no longer speculative. When an instruction is no longer speculative, we allow it to update the register file or memory; we call this additional step in the instruction execution sequence instruction commit. 

Using the bypassed value is like performing a speculative register read because we do not know whether the instruction providing the source register value is pro- viding the correct result until the instruction is no longer speculative. When an instruction is no longer speculative, we allow it to update the register file or mem- ory; we call this additional step in the instruction execution sequence instruction commit.

The key idea behind implementing speculation is to allow instructions to exe- cute out of order but to force them to commit in order and to prevent any irrevo- cable action (such as updating state or taking an exception) until an instruction commits. Therefore, when we add speculation, we need to separate the process of completing execution from instruction commit, because instructions may finish execution considerably before they are ready to commit. Adding this commit phase to the instruction execution sequence requires an additional set of hardware buffers that hold the results of instructions that have finished execution but have not com- mitted. This hardware buffer, which we call the reorder buffer, is also used to pass results among instructions that may be speculated.


### Multiple Issue and Static Scheduling
Perviously mentioned techniques help eliminate data, control stalls and achieve an idela CPI of 1. To improve performance further, we want to decrease the CPI to less than one, but the CPI cannot be reduces below one if we issue only one instruction every clock cycle.

_Multiple-issue processors_ come in 3 flavors:
  - Statically scheduled superscalar processors (varying number of instructions per clock, in-order execution)
  - VLIW (very long instruction word) processors (fixed number of instructions per clock, statically scheduled by compiler)
  - Dynamically scheduled superscalar processors (varying number of instructions per clock, out-of-order execution)



### Helpful Links
- https://www.cs.umd.edu/~meesh/411/CA-online/chapter/dynamic-scheduling-loop-based-example/index.html
