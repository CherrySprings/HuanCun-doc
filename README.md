# HuanCun-doc

English translation for XiangShan L2/L3 cache [documentation](https://xiangshan-doc.readthedocs.io/zh_CN/latest/huancun/overview/).

## Architecture

The L2/L3 Cache ([Huancun](https://github.com/OpenXiangShan/HuanCun) subproject) of XiangShan Nanhu processor is a directory-based non-inclusive cache (inclusive directory, non-inclusive data) designed with reference to [block-inclusive-cache-sifive](https://github.com/sifive/block-inclusivecache-sifive).

Huancun uses Tilelink as the bus consistency protocol, and can solve the cache alias problem when the L1 Cache is larger than 32KB by adding a custom Tilelink user-bit.

The overall structure of huancun is shown in the figure below:

![HuanCun](https://xiangshan-doc.readthedocs.io/zh_CN/latest/figs/huancun.png)

Huancun divides the data storage into slices with indices according to the low bits of the requested address to improve concurrency. The number of MSHRs inside each slice is configurable and is responsible for specific task management.

DataBanks are responsible for storing specific data, and the number of banks can be configured through parameters to improve the concurrency of reading and writing.

RefillBuffer is responsible for temporarily storing refilling data, so as to directly bypass to the upper layer cache without writing through SRAM.

The Sink / Source related module is the Tilelink channel control module, responsible for interacting with the standard Tilelink interface. On the one hand, it converts external requests into cache internal signals, and on the other hand, receives cache internal requests and converts them into Tilelink requests sent to the interface.

In terms of directory organization, huancun stores the upper-level data and the current-level data directory separately. Self Directory / Client Directory are the directory corresponding to the current level cache data and the directory corresponding to the upper level cache data.

In addition, the prefetcher adopts the BOP (Best-Offset Prefetching) algorithm, which can be configured or adjusted by parameters.

## Directory Design

This chapter will introduce the design of directories in huancun. The "directory" referred to in this chapter is broad and includes metadata and tags.

Huancun is a non-inclusive cache based on directory structure, inspired by NCID (Zhao, Li, et al. "NCID: a non-inclusive cache, inclusive directory architecture for flexible and efficient cache hierarchies." Proceedings of the 7th ACM international conference on Computing frontiers. 2010.) during the design process. Between multi-level caches, data is non-inclusive, while directories are inclusive. In terms of structural organization, huancun stores the upper-level data directory and the local data directory separately, and the two structures are similar. The directory is built using SRAM, each set accessed by index, containing #Way lines of data.

Each item in the local data directory stores the following information:

- state: saves the permission of data block (one of Tip / Trunk / Branch / Invalid)

- dirty: indicates whether the data block is dirty

- clientStates: saves the permission status of the data block in the upper layer (only meaningful when the block is not Invalid)

- prefetch: indicates whether the data block is prefetched

Each item in the upper data directory stores the following information:

- state: saves the permission of the upper layer data block

- alias: save the last bit of the virtual address of the upper layer data block (that is, alias bit, see cache alias problem for details)

### Directory Read

When the MSHR Alloc module allocates a request into MSHR, it also initiates a read request to the directory at the same time. It reads the upper-layer and local metadata and tag in parallel, judges whether it is a hit according to tag comparison, and transfers the metadata of the corresponding way to the corresponding MSHR.

When it is a hit in the directory, the passed way is the hit way; when it is a miss, the passed way is the way obtained according to the replacement algorithm. The replacement algorithm can be flexibly configured. The local data directory of the Nanhu version uses the PLRU replacement algorithm, and the upper data directory uses a random replacement algorithm.

### Directory Write

When the MHSR transaction is reaching the end, it is usually necessary to write the directory to update the status, tag and other information. The directory has 4 write request ports, which receive the writes of local metadata, upper-layer metadata, local tag, and upper-layer tag respectively. Write requests have priority over read requests.

In terms of connections, the above four write requests of all MSHRs are separately arbitrated into the directory. The arbitration priority relationship is carefully designed to avoid request nesting errors or deadlocks.

### FAQs and Design Considerations

**There is already upper-level metadata in the directory, why does clientStates still exist in the local metadata?**

When a request, such as Acquire BLOCK_A, is missed in the local directory, the directory will select a path of information to pass to MSHR according to the replacement algorithm. At this time, this path may not be invalid, but there is a data block BLOCK_B. In order to complete this Acquire request, we need to know the state information of BLOCK_B in the upper layer. In order to avoid reading the directory twice, we save an additional copy of clientStates in the local metadata.

**Is there a read-write race hazard for the directory?**

Our MSHR is blocked according to sets, and new requests will enter and read the directory only after the MSHR is released, so there will be no risk of read-write race hazard.

## Data Storage

The `DataStorage` module is responsible for the storage, reading and writing of cache data. According to the characteristics of each Tilelink channel, the module has 2 read ports (`sourceC_r`, `sourceD_r`) and 3 write ports (`sinkD_w`, `sourceD_w`, `sinkC_w`). In order to improve the read-write concurrency, the The module can parameterize and configure the number of internal banks, and different banks can read and write in parallel. In addition, the module also supports parameterized pipeline for the data read from the SRAM, so as to meet higher frequency requirements.
