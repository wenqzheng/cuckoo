Cuckoo Cycle
============
Blog article explaining Cuckoo Cycle at
http://cryptorials.io/beyond-hashcash-proof-work-theres-mining-hashing

Whitepaper at
https://github.com/tromp/cuckoo/blob/master/doc/cuckoo.pdf?raw=true

Cuckoo Cycle is the first graph-theoretic proof-of-work,
and the most memory bound, yet with instant verification.

Proofs take the form of a length 42 cycle in a bipartite graph with N nodes and
N/2 edges, with N scalable from millions to billions and beyond.

The graph is defined by the siphash-2-4 keyed hash function mapping an edge index
and partition side (0 or 1) to the edge endpoint on that side.
This makes verification trivial: hash the header with blake2b to derive the siphash key,
then compute the 42x2 edge endpoints with 84 siphashes, check that each
endpoint occurs twice, and that you come back to the starting point only after
traversing 42 edges (this also makes Cuckoo Cycle, unlike Hashcash, relatively
immune from Grover's quantum search algorithm).

A final blake2b hash on the sorted 42 nonces can check whether the 42-cycle
meets a difficulty target.

This is implemented in under 220 lines of C code (files src/{siphash.h,cuckoo.h,cuckoo.c}).

From this point of view, Cuckoo Cycle is a very simple PoW,
requiring hardly any code, time, or memory to verify.

Finding a 42-cycle, on the other hand, is far from trivial,
requiring considerable resources, and some luck
(for a given header, the odds of its graph having a L-cycle are about 1 in L).

The memory efficient miner uses 3 bits per edge and is bottlenecked by
accessing random 2-bit counters, making it memory latency bound.  The roughly
4x faster latency avoiding miner, a rewrite from xenoncat's bounty winning solver,
uses 33 bits per edge and is bottlenecked by bucket sorting. making it memory bandwidth bound.

Hybrid ASICs
------------
Its large memory requirements make single-chip ASICs economically infeasable for Cuckoo Cycle.
For the default billion node graph size, the bandwidth bound solver needs well over 2GB,
currently requiring a multitude of 1GB DRAM chips.
DRAM can be viewed as an Integrated Circuit Customized to the Application of writing and reading words of memory
in mostly sequential fashion. It's perhaps the most cost optimized and commoditized ASIC in existence.
It uses only moderate power (on the order of 1W per chip) and is ubiquitous in the device landscape.
Every modern smart phone includes a few GBs of DRAM that mostly sits idle as it recharges overnight.
This presents unique opportunities for a PoW that is minimally compute intensive and maximally memory intensive.

A hybrid ASIC solution for Cuckoo Cycle pairs a bunch of DRAM chips with a small low-power ASIC,
which needs to run just efficient enough to saturate the limited DRAM bandwidth.
In terms of solutions per Joule of energy, this might be reasonably efficient mining platform.
Adding such chips to existing devices, using already present DRAM as a sunk cost, could make for a
cost effective mining platform.

An indirectly useful Proof of Work
--------------
Although running the latency bound solver requires an order of magnitude less memory,
current low-latency memory ASICs such as RLDRAM3 and QDR-IV SRAM are at least an order of magnitude more expensive.
Cuckoo Cycle provides incentives to making low-latency memory affordable enough to tip the scale,
which could benefit many applications beyond mining.

Cycle finding
--------------
The algorithm implemented in lean_miner.hpp runs in time linear in N.
(Note that running in sub-linear time is out of the question, as you could
only compute a fraction of all edges, and the odds of all 42 edges of a cycle
occurring in this fraction are astronomically small).

Memory-wise, it uses N/2 bits to maintain a subset of all edges (potential
cycle edges) and N additional bits to trim the subset in a series of edge trimming rounds.
The bandwidth bound algorithm implemented in mean_miner.hpp instead uses 16N bits to maintains
lists of edges and less than N bits for trimming.
This is the phase that takes the vast majority of runtime.

Once the subset is small enough, an algorithm inspired by Cuckoo Hashing
is used to recognise all cycles, and recover those of the right length.

Performance
--------------

The runtime of a single proof attempt for a 2^30 node graph on a 4GHz i7-4790K is 10.5 seconds
with the single-threaded mean solver, using 2200MB (or 3200MB with faster solution recovery).
This reduces to 3.5 seconds with 4 threads (3x speedup).

Using an order of magnitude less memory (just under 200MB),
the lean solver takes 32.8 seconds per proof attempt.
Its multi-threading performance is less impressive though,
with 2 threads still taking 25.6 seconds and 4 taking 20.5 seconds.

I claim that siphash-2-4 is a safe choice of underlying hash function,
that these implementations are reasonably optimal,
that trading off (less) memory for (more) running time,
incurs at least one order of magnitude extra slowdown,
and finally, that mean_miner.cu is a reasonably optimal GPU miner.
The latter runs about 2.4x faster on an NVIDA 1080Ti than mean_miner on an Intel Core-i7 CPU.
In support of these claims, I offer the following bounties:

CPU Speedup Bounties
--------------------
$10000 for an open source implementation that finds 42-cycles twice as fast
as lean_miner, using no more than 1 byte per edge.

$10000 for an open source implementation that finds 42-cycles twice as fast
as mean_miner, regardless of memory use.

Linear Time-Memory Trade-Off Bounty
-----------------------------------
$10000 for an open source implementation that uses at most N/k bits while finding 42-cycles up to 10 k times slower, for any k>=2.

All of these bounties require N ranging over {2^28,2^30,2^32} and #threads
ranging over {1,2,4,8}, and further assume a high-end Intel Core i7 or Xeon and
recent gcc compiler with regular flags as in my Makefile.

GPU Speedup Bounty
------------------
$5000 for an open source implementation for a consumer GPU
that finds 42-cycles twice as fast as mean_miner.cu on 2^30 node graphs on comparable hardware.

The Makefile defines corresponding targets leancpubounty, meancpubounty, tmtobounty, and gpubounty.

Double and fractional bounties
------------------------------
Improvements by a factor of 4 will be rewarded with double the regular bounty.

In order to minimize the risk of missing out on less drastic improvements,
I further offer a fraction FRAC of the regular CPU/GPU-speedup bounty, payable in bitcoin cash,
for improvements by a factor of 2^FRAC, where FRAC is at least one-tenth.
Note that 2^0.1 is about a 7% improvement.

Siphash Bounties
----------------
While both siphash-2-4 and siphash-1-3 pass the [smhasher](https://github.com/aappleby/smhasher)
test suite for non-cryptographic hash functions,
siphash-1-2, with 1 compression round and only 2 finalization rounds,
[fails]https://github.com/tromp/cuckoo/doc/SipHash12) quite badly in the Avalanche department.
We invite attacks on Cuckoo Cycle's dependence on its underlying hash function by offering

$5000 for an open source implementation that finds 42-cycles in graphs defined by siphash-1-2
twice as fast as lean_miner on graphs defined by siphash-2-4, using no more than 1 byte per edge.

$5000 for an open source implementation that finds 42-cycles in graphs defined by siphash-1-2
twice as fast as mean_miner on graphs defined by siphash-2-4, regardless of memory use.

These bounties are not subject to double and/or fractional payouts.

Happy bounty hunting!
 
How to build
--------------
<pre>
cd src
export LD_LIBRARY_PATH="$PWD:$LD_LIBRARY_PATH"
make
</pre>

Bounty contributors
-------------------

<ul>
<li> <a href="https://forum.z.cash/">Zcash Forum</a> participants</li>
<li> <a href="https://www.genesis-mining.com/">Genesis Mining</a> </li>
<li> <a href="https://www.simply-vc-co.ltd/?page_id=8">Simply VC</a> </li>
<li> <a href="https://bitcointalk.org/index.php?topic=1670733.0">Claymore</a> </li>
<LI> <a href="http://www.aeternity.com/">Aeternity developers</a>
</ul>

Projects using, or planning to use, Cuckoo Cycle
--------------
<UL>
<LI> <a href="https://github.com/mimblewimble/grin">Minimal implementation of the MimbleWimble protocol</a>
<LI> <a href="http://www.aeternity.com/">æternity - the oracle machine</a>
<LI> <a href="https://github.com/bitcoin/bips/blob/master/bip-0154.mediawiki">BIP 154: Rate Limiting via peer specified challenges; Bitcoin Peer Services</a>
<LI> <a href="http://www.raddi.net/">Raddi // radically decentralized discussion</a>
<LI> <a href="https://bitcointalk.org/index.php?topic=2360396">[ANN] *Aborted* Bitcoin Resilience: Cuckoo Cycle PoW Bitcoin Hardfork</a>
</UL>

![](img/logo.png?raw=true)
