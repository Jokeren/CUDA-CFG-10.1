# CUDA-CFG-10.1

We use CUDA-10.1.

We use the following command to dump cfgs:

    nvdisasm -cfg -poff <cubin_name>

Stream\_COPY.cfg is the cfg of the unoptimized (-O0) Stream\_COPY kernel in RAJAPerf Suite.

tensor\_contraction.cfg is the cfg of the optimized (-O3) tensor\_contraction kernel in ExaTensor.

### Problem 1

There are some dangling blocks not connected with other blocks in cfgs.

I am okay with the current representation because the problem is really with boost APIs. 

In tensor\_contraction.cfg, `.L22` is a dangling block. The block is supposed to be linked by `.L_11`. It makes sense to separate this block on the CFG because the instruction before it is an EXIT instruction, meaning that the block is never executed.

It is not only the last block has the chance to become a dangling block. In Stream\_COPY.cfg, `.L_15` and `.L_14` are dangling blocks, while `.L_15` is the last block, `.L_14` is a internal block.

Our control flow analyzer uses boost graphviz APIs to parse a dot graph, which does not reflect subgraphs as actual entities. The reader API returns a set of nodes in the dot graph without identifiying which subgraph each node belongs to. Therefore, if there are several dangling blocks with the same instruction addresses, we couldn't associate them with the correponding subgraph. Currently, we just fill in NOP instructions for these dangling blocks.

If nvdisasm could link these dangling blocks, we can construct accurate control flow graphs, If not, it is still fine because in most cases instructions in these blocks are not important.

### Problem 2

Blocks in the CFGs are not "basic blocks".

https://en.wikipedia.org/wiki/Basic_block

    In compiler construction, a basic block is a straight-line code sequence with no branches in except to the entry and no branches out except at the exit.

We notice that in tensor\_contraction.cfg, however, blocks are not diviced by branch instructions. Instead, they are grouped by some unknown rules. 

In comparison, in "Stream\_COPY.cfg", most blocks end with branch instructions. Therefore, I supppose it is the compiler that merges blocks together such that the final cfg in the optimized version.
