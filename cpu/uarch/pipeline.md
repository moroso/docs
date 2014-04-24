# Pipeline microarchitecture #

## Introduction ##

A valuable concept to think about is the question of what causes a stall
(i.e., what causes no further instructions to be issued).  For the purposes
of this brief introduction, I'll discuss only memory-related stalls, but of
course there can be other, more mundane, pipeline stalls also.  I focus on
memory-related stalls because they are not only the most variable in
latency, but they can be the most substantial in latency, too: a local cache
hit may have a latency of just a few cycles, but going out to main memory
could be as high as 30 cycles.

The most obvious scheme is to simply hide the minimum latency in the
pipeline, and then match the pipeline lengths for all execution units.  By
doing this, all execution units operate in lock-step, and in the "best
case", memory results come out when they are ready.  The downside is that if
that minimum pipeline length is longer than the minimum pipeline length for
integer operations, then (without forwarding, at least), integer operations
now grow in latency -- and additionally, on memory loads that take a long
time, then the entire pipeline stops, even if other useful work can be done.

Other nominally in-order architectures get around this, such as TILEPro, get
around this with the concept of "stall-on-use".  (See section 4.2.1 of the
[TILEPro architecture manual][tilearch].)  In that case, an instruction only
fails to issue when it is waiting for a memory load that has not yet
returned; a register that has not yet written simply stalls.  This decouples
the integer pipeline; additionally, it allows the memory pipeline to hide
latency (for instance, multiple fetches can be issued while waiting for the
first to complete).

## Bad idea ##

(This is aggressively summarized.  Don't worry if it doesn't make much
sense.  It doesn't work.  The concepts within are explained better later.)

In the shower, I had a good idea that actually was a bad idea.  The basic
concept was that to maintain architectural consistency (i.e., be able to
have precise exceptions), we'd have a FIFO full of expected register
results: each time we issue, we push to the FIFO with what registers we
expect to write to later.  Then, we declare architectural commit to have
happened when all four pipelines that are expected to return a value do. 
(The pipelines also have FIFOs so that they do not have to stall waiting for
other pipelines to complete.)

The reason why this works poorly is that everything becomes backlogged.  If
I issue a load, and then go off and do some arithmetic in the background,
none of the arithmetic can commit -- even if it doesn't depend on the load!
-- because it needs to be ordered behind the load.  So, as a result, if the
arithmetic depends on intermediate results, it will be stalled at issue
waiting for the load to commit anyway.

This very neatly defeats the purpose of stall-on-use.  Basically, the only
thing this is good for is writing a fast memcpy(), in which you do a bunch
of loads, have no dependencies, and then do a bunch of work on the loads
long enough later that you have evaded the stall.

## Repairing the bad idea ##

First off, some terminology: architectural state is defined as the
user-visible things.  For instance, the register file and program counter
are the architectural state.  A precise exception is one in which the
architectural state is as it was at some particular pc address; this is
distinguished from an imprecise exception, in which the architectural state
after an exception may not correspond to that from any single pc.  Imprecise
exceptions are generally not possible to recover from.  An architectural
commit point is the point in the pipeline at which an instruction that makes
it across is guaranteed to make it to a precise architectural state.

One solution that I was thinking of was to introduce a register reorder
buffer, to allow 'temporary commits', and then roll back to a known valid
architectural state if we abort at a previous program counter address.  This
is the classic solution in a Tomasulo-like system.  It would work, but the
effort of implementing it would be very high, and at that expense, it may be
worth simply implementing the rest of a Tomasulo out-of-order machine.  This
is unpleasant, to be sure.

The obvious solution that I was missing was the solution that Tile64 used. 
Tile solved this by simply moving the architectural commit point earlier in
the pipeline!  In my proposed scheme above, the commit point is exactly when
the register results are ready; in the event of an exception, we can stop
execution immediately, drain the FIFOs, and simply roll the pc back to the
last committed pc.

In the correct scheme, though, we simply place the commit point just after
where the last opportunity to generate an exception is -- a SYSCALL
instruction for control, or after the TLB could generate a miss -- and then
allow instructions to commit in any order after that.  (Of course, we track
which registers are alive, to appropriately stall for read-after-write
hazards.  We also need to take appropriate care to stall for
write-after-write hazards in issue.)  Then, in the case of an exception or
interrupt, we need to wait for all outstanding register writebacks to drain,
and then the last updated pc is the last bundle that passed through the
commit point.

In this new scheme, we can also remove the commit-pc FIFO that tracks which
bundles still are waiting for data to commit.  So, with appropriate care,
this scheme should prove more efficient, and about as easy to implement.

[tilearch]: http://www.tilera.com/scm/docs/UG120-Architecture-Overview-TILEPro.pdf "TILEPro Architecture Overview"