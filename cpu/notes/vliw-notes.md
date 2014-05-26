# Some thoughts about the VLIW aspects of the ISA #

We will have 4 "lanes" that execute instructions simultaneously.

* Lane 0 can execute any instruction.
* Lane 1 can execute ALU or memory operations.
* Lane 2 can execute only ALU operations.
* Lane 3 can execute only ALU operations without long immediates.


With our current single-instruction encoding (encoding-jackieh seems to be the foremost candidate at the moment) it doesn't seem like we can save any meaningful amount of encoding space with the limited capabilities of the lanes - the ALU operations utilize basically all of the 32 bits on operands. Therefore, instructions will be organized into size-aligned 128-bit "packets" consisting of a 32-bit instruction for each lane. 

If an ALU instruction specifies a long immediate operand, the next 32 bits of the packet are interpreted as the immediate operand for that instruction, and a NOP is issued on the next lane. The last lane is incapable of processing instructions with long immediates; this does not restrict the combinations of instructions that can be executed simultaneously.

The following is undefined behavior:
* Attempting to execute an instruction on a lane that does not support that instruction type.
* Specifying the same destination register for more than one instruction in a packet.
* Specifying two accesses to the same (physical) memory location in the same packet. Two reads is fine.

We need some sane definition of "undefined" here to allow the kernel to maintain its security and separation guarantees in the face of user code running undefined instructions. The following are apparently reasonable behaviors for the three cases above:

* Execute some undefined but valid instruction instead (also reasonable if an instruction uses one of the reserved opcodes)
* Corrupt the destination register (or possibly other registers, too!)
* Corrupt the memory location being written to (or possibly the whole cache line around it, but not some random other part of memory)

If we can't figure out a way to ensure nothing bad will happen, or if we have the hardware budget for the extra logic in the decoder, we might raise an invalid instruction exception in either of the first two cases.