# Straw Proposal for the Moroso CPU #

Here's a straw proposal for an ISA.  One of the directions that we can go in is tightly coupling the ISA to the internal architecture -- this saves decode effort in the CPU, and if we don't care about forward-compatibility, then we really get good simplicity benefits with relatively little losses.

The "easiest possible thing" is MIPS R4000.  A MIPS is very easy to build, and is also not very area-intensive.  It's also not terribly performant: an R4k is an in-order, single-issue machine.  But, we can augment it in a handful of ways to make it more performant.  (As a consequence, the machine also becomes more "interesting".)

The basic premise for things that we want to add is twofold -- and irritatingly, both four-letter acronyms:

* **SIMD (Single Instruction Multiple Data).**  A SIMD machine has instructions that operate on wide registers, performing a handful of operations in parallel.  We want it because it means that we can share control resources, and devote more area to doing actual useful work.
* **VLIW (Very Long Instruction Words).**  A VLIW machine, in general, operates not on instructions, but instruction packets.  In a classic superscalar (i.e., capable of issuing more than one instruction per clock) machine, the processor has to do the work of understanding which instructions can be parallelized, and scheduling them to execution units.  In a VLIW machine, more of that work is offloaded to compile-time, and the processor is handed "packets" of multiple instructions tied together.  The downside of this is that without careful design, instruction efficiency drops (i.e., more memory bandwidth is devoted to instruction fetch), and that the instruction encoding is intimately tied to the internal architecture of the machine.  We may care about the former (though we may be able to design around it), but we probably care very little about the latter (forward compatibility is probably not a goal).

So, the system that I propose is something like a VLIW-MIPS.  Major architectural decisions about instruction set are taken primarily from MIPS32, but it is augmented to add the above features.  In particular, the system looks something like this:

* The machine is four-wide superscalar, in-order, with 128-bit instruction packets.
* There are four classes of instruction:
  * C-type (control) -- branch, call, return, coprocessor
  * L-type (load/store)
  * I-type (integer)
  * S-type (SIMD) -- needs definition
* Four execution units:
  * #1 supports all four classes of instruction (is there encoding space to do that?)
  * #2 supports L-type, I-type, S-type
  * #3 supports I-type, S-type
  * #4 supports I-type, S-type
* Upshot: register file is eight-read, four-write
  * 32 registers, like MIPS
  * Maybe segregate SIMD registers?
* To more easily load literals, you can "steal" the adjacent execution unit's slot in the instruction packet
* P1 feature: variable length instruction packets, to increase encoding efficiency
* Soft-loaded TLB; whole memory range is TLB-mapped (we get rid of MIPS's silly KUSEG/KSEG0/KSEG1/KSEG2 business)
* Instruction predication available (on each instruction in a packet), because otherwise things get pretty expensive in a VLIW machine
  * Probably two or more predicate registers?  Consider Qualcomm Hexagon for inspiration: http://pages.cs.wisc.edu/~danav/pubs/qcom/hexagon_hotchips2013.pdf
The upshot of this for Team Compiler is:
* You will need to have your own instruction scheduler
  * (this can be in its own pass -- the thing I recommend is to generate a linear instruction stream, then run it through the scheduler, like a CPU would do)
* You should get reading about compiling for predicated machines

## Other SoC bits ##

Peripherals --
* DMA for talking to the peripherals

Bus interface --
* which RAMs are we using?

Board --
*  FPGA boards with DRAM get expensive -- Atlys has 128MB of 16-bit wide DDR2-800 (ugh!  that's 1.6 GByte/sec, which is pretty pitiful), and is $400; Nexys 4 hasn't got any DRAM at all!
* A good option: http://www.terasic.com.tw/cgi-bin/page/archive.pl?Language=English&No=830
