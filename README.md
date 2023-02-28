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

## MSHR Design

This chapter introduces the design of non-inclusive MSHR (Miss Status Holding Registers) in Huancun.

When Huancun receives an Acquire/Release request from the upper cache, or a Probe request from the lower cache, it will assign an MSHR to the request, and at the same time obtain the permission information of the address in the cache at this layer and above by reading the directory. Based on this information it decides:

- Permission control: how to update the permissions in self directory/client directory

- Request control: whether it is necessary to send sub-requests to the lower-level cache and wait for the response of these sub-requests

The following sections will take L2 Cache as an example to introduce the design of huancun MSHR.

### Permission Control

The data in huancun is stored non-inclusively, but the directory strictly adopts the inclusion policy, so MSHR can get all the permission information of the request address in ICache, DCache and L2 Cache, so as to control the change of the permission of the address. Each address in XiangShan’s cache system follows the rules of the TileLink consistency tree, and each block has four states: N (None), B (Branch), T (Trunk), and TT (TIP) in each layer of the cache system. The first three correspond to no permission, read-only, readable and writable respectively. The consistency tree grows from bottom to top in the order of memory, L3, L2, and L1. As the root node, memory has readable and writable permissions. In each layer, the permissions of child nodes cannot exceed the permissions of the parent node. Among them, TT represents the leaf node on the branch with T permission, indicating that the upper layer of the node only has N or B permission. On the contrary, the node with T permission instead of TT permission means that the upper layer must have T/TT permission node. For detailed rules, please refer to Chapter 9.1 of the TileLink manual.

MSHR updates the dirty bit, permission field, and clientStates field of the self directory according to the type of request and the result of reading the directory. clientStates indicates the permission of the requested address in the upper layer L1 cache if this address has permission in the current cache (B and above permission). In addition, MSHR will also update the client directory corresponding to ICache and DCache, including the permission domain and alias domains (covered in the chapter of cache aliasing issues).

Here are two examples to briefly explain how MSHR judges the modification of permissions, and why these fields are needed in the design of the directory.

1. Suppose DCache has the permission of address X, but ICache does not, then ICache requests the permission of X from L2. At this time, L2 needs to judge whether L2 has block X, whether DCache has block X, and if so, whether the permission is sufficient, etc. MSHR will decide whether to return data directly to ICache or ask DCache for data (Probe) and then return it to ICache still needs to forward the L3 Acquire block to ICache, so we need to maintain permission bits in the self directory and client directory respectively.

1. Suppose DCache has the permission of address X, but L2 does not, then DCache replaces X and sends a request to release address X. L2 allocates an MSHR after receiving the request, and we hope that L2 can save the block X of DCache Release. If it needs to be replaced, L2 will also release block Y to L3 according to the replacement policy. In this example, we need to know whether Y is dirty data in L2, which involves whether data needs to be carried when releasing Y to L3, so the dirty bit is required in self directory. When replacing Y out of L2, we need to know that Release should carry what kind of parameters, so the clientStates domain needs to be maintained in the self directory, so as to judge the permission of the upper-layer node on address Y.

More details about directories can be found in the chapter of directory design.

请求控制
MSHR需要根据请求的内容和读目录的结果判断需要完成哪些子请求，包括是否需要向下Acquire或Release，是否需要向上Probe，是否应该触发一条预取，是否需要修改目录和tag等等；除了子请求，MSHR还要记录需要等待哪些子请求的应答。

MSHR将这些要调度的请求和要等待的应答具体成一个个事件，并用一系列状态寄存器记录这些事件是否完成。s_*寄存器表示要调度的请求，w_*寄存器表示要等待的应答，MSHR在拿到读目录的结果后会把需要完成的事件(s_*和w_*寄存器)置为false.B，表示请求还未发送或应答还没有收到，在事件完成后再将寄存器置为true.B，当所有事件都完成后，该项MSHR就会被释放。

### Request Control

MSHR needs to judge which sub-requests need to be completed according to the content of the request and the result of reading the directory, including whether to Acquire or Release downward, whether to Probe upward, whether to trigger a prefetch, whether to modify the directory and tag, etc. Besides request, MSHR also needs to record the pending status for sub-request responses.

MSHR concretes these scheduled requests and pending responses into individual events, and uses status registers to record whether these events are completed. The `s_*` register indicates the request to be scheduled, and the `w_*` register indicates the response to be waited for. After MSHR gets the result of reading the directory, it will set the event to be completed (`s_*` and `w_*` register) to `false`, indicating that the request is still pending. If the event has not been sent or the response has not been received, set the register to `true` after the event is completed. When all events are completed, the MSHR will be released.

## TileLink Channel Control

This chapter introduces the structure of each channel control module in huancun. Readers need to be familiar with the TileLink bus protocol.

The channel control module is divided into a Sink module and a Source module.

- The Sink module receives active requests and passive responses on the TileLink bus. For active requests (SinkA, SinkB, SinkC), convert the request into a huancun internal request and send it to the MSHR Alloc module or Request Buffer; for passive responses, process the response and send it back to the corresponding MSHR (SinkD, SinkE)

- The Source module receives the internal request of MSHR, processes and packages it and sends it to the TileLink bus (SourceA, SourceB, SourceC, SourceD, SourceE)

In addition, some modules will additionally receive Tasks sent from MSHR to complete some related tasks.

### SinkA

SinkA forwards the request received from channel A to MSHR Alloc. If there is no data, forward directly. If there is data (such as PutData, etc.), it needs to be stored in PutBuffer. Each beat of the data is stored in the free entry in PutBuffer in the form of an array, and the index of the free item is forwarded to MSHR Alloc.

The data stored in PutBuffer will be used by SourceD or SourceA. When the Put request is hit in the cache, SourceD is responsible for writing it into DataStorage. When it is Miss, SourceA is responsible for directly forwarding it to the next level of cache or memory.

When receiving a new request, channel A will block because MSHR Alloc is not ready or PutBuffer is full (when there is data). Once a request with data is received, it will not block the reception of subsequent Beats.

### SinkB

SinkB forwards the request received from channel B to MSHR Alloc, without any other logics.

### SinkC

SinkC receives the request from the upper layer to release permissions or data, including the Release / ReleaseData actively released by the client and the response ProbeAck / ProbeAckData that is passively released, and forwards the request to MSHR Alloc.

The processing of ReleaseData / ProbeAckData is similar to the processing flow of SinkA's PutData request. SinkC maintains a buffer. Each Beat of the data is stored in a free entry in the buffer in the form of an array, and the index is forwarded to MSHR Alloc. SinkC receives task from MSHR to process the data in the buffer. Task is divided into three types:

- Save: Save the data items in the buffer into DataStorage

- Through: Pack the data items in the buffer and directly release them to the lower layer Cache or memory

- Drop: Discard data items in the buffer

When receiving a task, SinkC enters the Busy state and does not receive subsequent tasks until the task processing is completed. If data beats that the sask needs to process have not been received, it will block the progress (controlled by beatValsThrough / beatValsSave).

### SinkD

SinkD receives the Grant / GrantData and ReleaseAck responses of the D channel, uses the source field to query the MSHR to get the set and way, and returns a Resp signal to the MSHR when it receives the first or last beat. For the response with data, the data from Grant will be sent to DataStorage and RefillBuffer at the same time.

### SinkE

SinkE receives GrantAck of E channel, sends Resp to MSHR, without any other logics.

### SourceA

SourceA receives Acquire and Put requests from MSHR and forwards them through channel A. For a Put request, it will read data from PutBuffer and forward downwards.

### SourceB

SourceB receives the task from MSHR and sends the Probe request through the B channel. SourceB maintains a status register workVec internally. Each bit of workVec corresponds to a client that supports Probe. When workVec is empty, it can receive requests from MSHR, and mark all clients that need Probe in the workVec register. After that, it sends Probes to these Clients and clears the flag bits one by one.

### SourceC

SourceC receives the task from MSHR, sends read requests to DataStorage, stores the data in the queue after receiving, and sends it out from the C channel after dequeuing.

The reason for setting up a queue is that SourceC's pipeline does not have a blocking mechanism, and DataStorage does not have blocking and canceling functions. This requires that once a task is allowed to be received, it must be able to process it and enter the queue. Back pressure control is used here to ensure that even if the C channel is always blocked, the space in the queue can accommodate all items that may enter, including the impending SRAMLatency item in the pipeline plus the beatSize items obtained by the current Task to read dataStorage.

### SourceD

SourceD is the most complicated one in the channel control module. It has two main responsibilities: read data and send it back to the upper level through the D channel, and send write requests to DataStorage to process Put and other requests.

Stage 1 sends read requests to DataStorage or RefillBuffer according to the task information, and sets Busy bit to block new tasks. It only accepts new tasks after the request of last beat is sent.

Stage 2 first sends read requests to the PutBuffer of SinkA, and stores the data in the queue after receiving the data. If the task does not need data or the data is bypassed from the RefillBuffer, this stage sends responses directly through the D channel.

Stage 3 is connected to stage 2 through a pipe, that is, the DataStorage returns the data before entering this stage. For general requests, this stage sends the read data through the D channel to respond. For Put requests, this stage sends an AccessAck response when processing the first Beat.

Stage 4 sends write requests to DataStorage.

**Bypassing Check**

For the same request, SourceD must have the lowest priority. The priority of each channel in DataStorage is actually determined according to the task priority of the same request. However, since MSHR will be freed before the request to SourceD is completed, the priority rules in DataStorage will be broken. For the Acquire request, it will be released after receiving the GrantAck, but the GrantAck of the client can be given when it receives the Grant first, then before the Grant last, MSHR will release it after receiving the GrantAck. For Get requests, GrantAck is not required. As long as SourceD is sent, MSHR will be released. In the case of these early frees, when Get and Acquire may still be reading data from SourceD, the next request for the same Set will come and issue a write operation to this set. From the perspective of correctness, we must first process the previous read, but SourceD has the lowest priority in DataStorage. The solution is to check the set/way of SourceD and compare it with the channel to be written. If they are the same, block them and let SourceD does it first.

### SourceE

SourceE receives the request from MSHR and sends GrantAck from E channel.

## L2 Prefetcher Design

XiangShan's L2 Cache uses Best-Offset hardware prefetch. BOP is an offset-based prefetcher. The offset can be dynamically adjusted according to the dynamic execution of the program. Its main goal is to ensure the timeliness of prefetching.

For the specific prefetching algorithm, please refer to the paper. We only introduce the parts related to the specific implementation of XiangShan.

First, a prefetch bit is used in the L2 directory to record whether a cache block is a prefetched block. When MSHR receives DCache's Acquire request, if the request address is not hit in L2 or hits the prefetched block, MSHR will initiate a "trigger prefetch" request, which will be sent to the prefetcher. The prefetcher will generate the prefetch address based on the request address plus the best offset trained by the best-offset algorithm, and then send the prefetch request to the bank where the address is located. The request type is Intent, and the MSHR allocator allocates an MSHR for the prefetch request. If the prefetch block is not in L2, MSHR will fetch the prefetch block from L3.

When MSHR completes a prefetch, it will send another response to the prefetcher, and the prefetcher will train the best-offset algorithm based on this response.

## Cache Aliasing Issue

Since both DCache and ICache use 128KB 8-way set associative structure in the NanHu architecture, the number of bits occupied by the cache index and block offset has exceeded the page offset (the page offset of a 4K page is 12 bits). This introduces the cache alias problem: when two virtual pages are mapped to the same physical page, the alias bits (alias bits) of the two virtual pages are likely to be different. If no additional processing is done, with VIPT (Virtual Index Physical Tag), the two virtual pages will be located in different sets of the cache, causing two copies of the same physical page to be cached in the cache, causing some consistency errors.

![alias1](https://xiangshan-doc.readthedocs.io/zh_CN/latest/figs/huancun_cache_alias-1.jpg)

In order to let L1 Cache continue to use VIPT, XiangShan's cache system uses hardware to solve the cache alias problem. The specific solution is that L2 Cache guarantees that a physical block has at most one alias bit in a VIPT cache in the upper layer.

Here is an example to illustrate how L2 solves the cache alias problem. As shown in the figure below, there is a block with a virtual address of 0x0000 in DCache, the virtual addresses 0x0000 and 0x1000 are mapped to the same physical address, and the aliases of these two addresses are different. At this time, DCache sends the address of 0x1000 block, and the alias (0x1) is recorded in the user field of the Acquire request. L2 finds that the request hits after reading the directory, but the alias of Acquire (0x1) is different from the alias (0x0) of DCache recorded by L2 at the physical address. So L2 will initiate a Probe sub-request, and record the alias (0x0) to be probed in the Probe data field. After the Probe sub-request is completed, L2 will return this block to DCache, and change the alias in the L2 client directory to is (0x1).

![alias2](https://xiangshan-doc.readthedocs.io/zh_CN/latest/figs/huancun_cache_alias-2.jpg)
