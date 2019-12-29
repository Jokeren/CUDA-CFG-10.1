# CUDA-CFG-10.1

We used CUDA-10.1.

We used the following command to dump CFGs:

    nvdisasm -cfg -poff <cubin_name>

*Stream\_COPY.cfg* is the CFG of the unoptimized (-O0 -G) *Stream\_COPY* kernel in RAJAPerf Suite.

*tensor\_contraction.cfg* is the CFG of the optimized (-O3 -lineinfo) *tensor\_contraction* kernel in ExaTensor.

### Problem 1

There are some dangling blocks not connected with other blocks.

We are okay with the current representation because the problem is really with boost APIs. 

In `tensor\_contraction.cfg`, `.L22` is a dangling block. The block is supposed to be linked by `.L_11`. It makes sense to separate this block from other blocks because the instruction before it is an EXIT instruction, meaning that the block is never executed.

It is not only the last block has the chance to become a dangling block. In Stream\_COPY.cfg, `.L_15` and `.L_14` are dangling blocks. While `.L_15` is the last block, `.L_14` is an internal block.

Our control flow analyzer uses boost graphviz API to parse a dot graph. The API returns a set of nodes in the dot graph without identifying which subgraph each node belongs to. Therefore, if there are several dangling blocks with the same instruction address, we cannot associate them with the corresponding subgraph, resulting in instruction "gaps." Currently, we just fill in NOP instructions for these gaps.

If nvdisasm links these dangling blocks in the output CFGs, we can construct accurate control flow graphs. If not, it is still fine because instructions in these blocks are mostly not important.

### Problem 2

Blocks in the CFGs are not "[basic blocks](https://en.wikipedia.org/wiki/Basic_block)."

    In compiler construction, a basic block is a straight-line code sequence with no branches in except to the entry and no branches out except at the exit.

In `tensor\_contraction.cfg`, however, blocks are not divided by branch instructions.

In comparison, in `Stream\_COPY.cfg`, most blocks end with branch instructions.

I suppose it is the NVCC compiler that merges basic blocks in the optimized version so that nvdisasm outputs "super" blocks in the CFGs.

We need the real basic block representation since our control flow analyzer identifies loop nests in CFGs. The loop analysis algorithm requires each basic block is a sequential code.
