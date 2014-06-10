# Stages of the core pipeline

## Instruction Fetch TLB Lookup
Looks up the instruction pointer address in the TLB and advances the instruction pointer. 
May have a branch predictor later, but let's say no at first. May take more than one cycle,
but should take one cycle on a TLB hit. Passes a physical address to the next stage.
This stage can generate page faults, in which case all downstream instructions must be flushed to the
commit point before the exception is handled.

## Instruction Fetch
Looks up the physical address in the instruction cache. May stall if that misses. 
Passes a 128-bit instruction packet to the decoder.

## Decode
This stage is responsible for instruction decoding and dependency resolution. Register arguments are 
also read from the register file at this stage.

We should decide if we're using four identical decoders or doing different things for the different
lane capabilities, as well as if we're generating exceptions for invalid instructions or instruction
types. If so we probably want to do that in the next stage, limiting exception sources to the fetch TLB
stage and the first execute stage.

## Execute Stage 1 - Exception / Branch Resolution
This is the first stage of the pipelined execution of any instruction type, during which branches and
exceptions are resolved. Exceptions can be generated in this stage by SYSCALL instructions, undefined
instructions, division by 0, or page faults on lanes 0 or 1. Each pipeline has its own exception cause
register, and lanes 0 and 1 have registers that store faulting addresses. An additional register stores
the address of the faulting instruction. 

Additionally, we can perform the first cycle's worth of work on instructions that don't generate
exceptions in this stage. My expectation is that ALU operations can actually complete in the same time
it takes for a TLB hit, so they should just take one cycle. Multi-cycle operations 
(e.g. multiply, divide) should be pipelined, with all but the first stage occuring after the commit
point.

It may make sense to have more than one pipeline stage before commit, to split up memory address
computation and TLB lookup. Some experimentation will be needed to determine the timing there.

This stage is executed independently by the four execution units, but is still done in lockstep
between all four lanes. TLB lookup may take variable amounts of time, but no lane can proceed to the
next stage before all four lanes have determined that there are no exceptions generated on this 
instruction.

## Execute Stage 2

Any remaining work to be done before register writeback is done in this stage, which may be pipelined
and take multiple cycles. At this point, each lane executes independently, and may take different
amounts of time to complete. I expect that for lanes 2 and 3, which only have ALUs, there will be no
work to be done after commit and writeback can occur immediately. However, lanes 0 and 1 have LSUs
here (taking physical addresses in from the previous stage), and lane 0 has the remainder of the 
miscellaneous opcodes (I expect multiplication and division to be the longest of those).

## Writeback Stage

Each lane has an independent writeback unit that writes results into the register file and clears the 
appropriate bit in the dependency scoreboard. Lanes 0 and 1 may have multiple results arriving at the 
register file from different execution units simultaneously, but should only write back a single 
register per cycle. This means there must be a simple arbitration policy in front of those - probably 
just a static priority to the longer instructions.

# Miscellaneous Other Modules

## L1 Instruction Cache
This is a read-only, single-ported cache connected to the instruction fetch stage in the pipeline by
a 128-bit-wide port. It connects to the LTC arbiter to fill in misses. It is physically-addressed.

## L1 Data Cache
This is a write-back, dual-ported cache connected to the LSUs on lanes 0 and 1 by 32-bit ports. It 
It connects to the LTC arbiter to fill in misses. It is physically-addressed.

## TLB
The TLB is 3-ported, and connects to the instruction fetch translation and stage and to lanes 0 and 1
in the pre-commit execute stage. It connects to the LTC arbiter for page table lookups, and must also
contain logic for hardware page table translation.

## Register File
32 32-bit registers. 8 reads, 4 writes per cycle. 3 1-bit predicate registers, plus one pinned to 1.

## Scoreboard
Keeps track of which registers or predicates are scheduled to be written. Bits are set by the decode
stage and cleared by the writeback units.
