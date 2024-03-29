# Pipelining

## What is pipelining?
Pipelining is muck like an assembly line. Because the processor works on different steps of the instruction at the same time, more instructions can be executed in a shorter period of time.


## Implementing a Pipeline for RISC V

### Unpiplined RISC V Instructions
Every RISC V instruction can be implemented in, at most, 5 clock cycles:
1. _Instruction fetch cycle_ (IF):
  ```
  IR <- Mem[PC]
  NPC <- PC + 4
  ```
  _Operation_ - Send out the PC and fetch the instruction from memory into the instruction register (IR); increment the PC by 4 to address the next sequential instruciton. The IR is used to hold the instrution that will be needed on subsequent clock cycles; likewise, the register NPC is used to hold the next sequential PC.

2. _Instruction decode/register fetch cycle_ (ID):
  ``` 
  A <- Regs[rs1];
  B <- Regs[rs2];
  Imm <- sign-extended immediate field of IR;
  ``` 
  _Operation_ - Decode the instruction and access the register file to read the registers (rs1 and res2 are the reigster specifiers). The outputs of the generatl-purpose registers are read intow two temporary registers (A and B) for use in later clock cycles. The lower 16 bits of the IR are also sign extended and stores into the temporary register Imm, for use in the next cycle.
  Decoding is done in paralel with reading reigsters, which is possible because these fields are at fixed location in the RISC V instruction format. Because the immedaite portion of a load and an ALU immediate is located in an identical place in every RISC V instructino, the sign-extended immediate is also calculated during this cycle in case it is needed in the next cycle. For stores, a separate sign-extension is needed, because the immediate field is split in two pieces.

3. _Execution/effective address cycle_ (EX):
  The ALU operates on the operands prepared in the prior cycle, performing one of four functions depending ont eh RISC V instruction type:
  - Memory reference:
    ```
    ALUOutput <- A + Imm;
    ```
    _Operation_ - The ALU adds the operands to form the effective address and places the result into the register ALUOutput.

  - Register-register ALU instruction:
    ```
    ALUOutput <- A func B;
    ```
  _Operation_ - The ALU performs the operation specified by the function code (a combination of the func3 and func7 fields) on the value in register A and on the value in register B. The result is placed in the temporary register ALUOutput.

  - Register-Immediate ALU instruction:
    ```
    ALUOutput <- A op Imm;
    ```
    _Operation_ - The ALU performs the operation specified by the opcode on the value in register A and on the value in register Imm. The result is placed in the temporary register ALUOutpu _Operation_ - The ALU performs the operation specified by the opcode on the value in register A and on the value in register Imm. The result is placed in the temporary register ALUOutput. 

  - Branch:
    ```
    ALUOutput <- NPC + (Imm << 2);
    Cond <- (A == B)
    ```
    _Operation_ - The ALU adds the NPC to the sign-extended immediate vlaue in Imm, which is shifted left by 2 bits to create a word offset, to compute the address of the branch target. Register A, which has been read in the prior cycle, is checked to determine wheter the branch is taken, by comparison with Register B, because we consider only branch equal. 
    The load-store architecture of RISC V means that effective address and execution cycles can be combined into a single clock cycle, because no instruction needs to simultaneously calcuate a data address, calculate an instruction target address, and perform an operation on the data. The other integer instructions not included herein are jumps of various forms, which are similar to branches.

4. _Memory access/branch completion cycle_ (MEM):<br>
  The PC is updated for all instruction: `PC <- NPC;`
  - Memory reference:
    ```
    LMD <- Mem[ALUOutput] or
    Mem[ALUOutput] <- B;
    ```
    _Operation_ - Access memory if needed. If the instruction is a load, data return from memory and are placed in the LMD (load memory data) register; if it is a store, then the data from the B register are written into memory. iIn either case, the address used is the one computed during the prior cycle and stored in the register ALUOutput.
  - Branch: 
    ```
    if (cond) PC <- ALUOutput
    ```
    _Operation_ - If the instruction branches, the PC is replaced with the branch destination address in the register ALUOutput. 
5. _Write-back cycle_ (WB):
  - Register-register or Register-immediate ALU instruction:
    ```
    Regs[rd] <- ALUOutput;
    ```
  - Load instruction:
    ```
    Regs[rd] <- LMD;
    ```
  _Operation_ - Write the result into the register file, whether it comes from the memory system (which is in LMD) or from the ALU (which is in ALUOutput) with rd designating the register. 

### Pipelined RISC V Instructions
We can pipeline the instructions with almost no changes by starting a new instruction on each clock cycle. Because every pipe stage is active on every clock cycle, all operations in a pipe stage must complete in 1 clock cycle and any combination of operations must be able to occur at once. Futhermore, pipelining it requires that values passed from one pipe stage to the next must be placed in registers, called _pipeline registers_ or _pipeline latches_.  They are IF/ID, ID/EX, EX/MEM, MEM/WB.

The process of letting an instruction move from the instruction decode stage (ID) into the execution stage (EX) of this pipeline is usually called instruction issue; an instruction that has made this step is said to have issued. For the RISC V integer pipeline, all the data hazards can be checked during the ID phase of the pipeline. If a data hazard exists, the instruction is stalled before it is issued.


### Dynamically Scheduled Pipelines
Simple pipelines fetch an instruction and issue it, unless there is a data dependence between an instruction already in the pipleine and the fetched instruction that can not be hidden with bypassing or forwarding.

Dynamic Scheduling is an apprach whereby the hardare rearranges the instruction execution to reduce the stalls. To implement _out-of-order_ execution (which implies out-of-order execution), we must split the ID pipe stage into two stages:
1. _Issue_ - Decode instructions, check for structural hazards. 
2. _Read opearnds_ - Wait until no data hazards, then read operands.

_Scoreboarding_ is a technique for allowing instructions to execute out of order when there are suffcicient resources and no data dependences. The goal of a scoreboard is to maintain an execution rate of one instruction per clock cycle (when no structural hazards are present) by executing an instruction as realy as possible. Thus, when the next instruction to execute is stalled, other instructions can be issued and executed if they do not depend on any active or stalled instruction.
The scoreboard takes full responsibility for instruction issue and execution, including all hazard detection.
