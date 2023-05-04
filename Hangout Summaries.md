# [24 Jan 2023] Intro to AOS and OS Structures

- Extremes of OS structures (generality vs specialization)

- - DOS-like: no protection between application and OS. Process abstraction does not exist in DOS. Hardware managed by OS. 
  - Monolithic: Each app in its own hardware address space. OS (services and device drivers) in its own hardware address space. Hardware managed by OS. 
  - Microkernel: Each app in its own hardware address space. Each OS service is in its own address space. Simple abstractions in microkernel (address space & IPC). 

- **Extensibility** of OS means the ability of customizing system services on the fly without taking down the system

- We can extend microkernel & DOS on the fly, but not monolithic OS. 

- **Protection domain** is an OS abstraction while **address space** is a hardware mechanism for implementing protection domain. 

- **Border crossing** => Going from one hardware address space to another. 



### Customized OS with SPIN

- - We cannot write entire OS is a type-safe high level language. 

  - Subsystems on top of SPIn (e.g. CPU scheduler) may have to step outside the boundaries of Modula-3 type safety. 

  - - Upon context switch (checkpoint/resume interface calls provided by SPIN CPU core services)
    - Populate processor registers from the context block of the process being resumed
    - To manipulate any physical hardware entity, must step out of modula-3 (all device drivers, interrupt handler, communication between globa scheduler and threads package)

  - Core services are trusted since they procide access to hardware mechanisms. These services may have to step outside the language enforced protection model to control the hardware resources. Applicatons have to trust what the kernel trusts. 

  - Writing entire OS in high level language is a myth. OS code does get dirty and ugly. We have to separate mechanisms and policies. 

  - SPIN OS is not secure. 

  - SPIN performance could be at best similar to monolithic

  - We cannot colocate user level processes within the extended SPIN kernel (so no border crossing at all). 

  - Create, resolve, and combine => End result created an OS with the same performance and protection attributes as monolithic

  - - Create: Create a logical protection domain
    - Resolve: Resolve the names that are implemented in one object file and used in another (like linking in compiler)
    - Combine: Combining different objects to create an aggregate domain

  - To pass up an event from SPIN to the library OS, there is no border crossing



# **[14 Feb 2023] Parallel Systems - Barrier, Communication, Scheduling, Case Studies**

#### Lock

- Anderson’s queue lock: false sharing as cache block can have multiple variables
- Linked list based queueing lock: size of queue will be equal # of processor

#### Barriers

- ###### Centralised Barrier/Counting Barrier

- - Sense reversing barrier: All except the last arrive: decrement count and spin on sense reversal. The last: reset the count to N, reverse sense

- ###### Tree Barrier: 

- - Advantages: less contention on shared variables (count and localsense). Less spinning compared to counting barriers. But depending on ariness of the tree multiple processors may still spin on same variable.
  - Disadvantages: Spin variables are not statically determined. P1 (if it arrives first) spins on level 1 localsense. P0 (arriving next) spins on level 2. Spinning on remote memory if NCC MP. Static spin location is important if the machine is NCC NUMA, then the processor will spin on a non-local memory causing contention on network.
  - Communication complexity of tree barrier: O(N)

- MCS Barrier does not require RMW atomic instruction. It works on CC and NCC shared memory machines

- Tournament Barrier

- - Spin location statically determined. No need fetch and phi operation
  - Communication can exploit parallel paths in ICN if available
  - Algo works with cluster machine (since can be done through message passing)
  - Compares to MCS, it does not exploit spatial locality of caches. MCS allocates multiple spleen varaibles in same cache. 
  - Works for both shared memory and message passing multiprocessors
  - Rounds of communication: Ceil(log2n); 
  - Total amount of communication: O(Nlog2N)

#### Communication (LRPC)

- RPC needs 2 traps, 2 context switches (C->S, S->C) for 1 procedure execution
- Copying overhead (4 in each direction): 1. Client stub marshals data as RPC message 2. Kernel copies RPC message into kernel buffer 3. Kernel copies into server domain 4. Server stub copies from server domain to server stack
- Using a A-stack to make RPC cheap => 2 copying (Copy from client stack to A-stack (shared); then server stub copy from A-stack to server stack.)

#### Scheduling Policies

- FCFS: Ignore affinity for fairness
- Fixed processor, Last processor (Focus on Cache affinity) -> Thread centric
- Minimum Intervening, Minimum intervening plus queue (focus on cache pollution) -> processor centric
- Inserting idle loop in processor scheduling may help improve system performance



# **[7 Mar 2023] Distributed Objects & Middleware**

##### Spring Approach

- Strong interfaces that are open, flexible, and extensible 
- Microkernel (threads/IPC) => follow Liedtke’s principle
- Use a network proxy (not part of the Spring kernel) to connect to other machines. 
- Spring uses object-oriented kernel: 1. Nucleus (threads & IPC) 2. Microkernel (nucleus + address space) 3. Door & Doortable 4. Object invocation and cross-machine call
- Tornado uses the clustered object as an optimization (incremental optimization of unchanging system services). In spring, the idea of an object pervades the entire system. They can make third-party providers able to write, write code that can be integrated into the operating system. 

##### Virtual Memory Management In Spring 

A region is a set of pages similar to linear address space is broken down into these different regions. Every region, as it is an abstraction called a memory object that backs a particular region of a virtual address space. A memory object maps to a physical thing (i.e. backing file on the disk, swap space). Virtual memory manager VMM is responsible for every address space. A pager object is responsible to serve a page fault in this region, it goes into the memory object and then pulls the page that it wants and populated it in the cache objects so that that process can have access to that virtual address space on which it had a page fault. We can have multiple pagers for different regions in the same address space. (Tornado one size fits all for paging). Pager object’s responsibility to keep the consistency of shared pages across VMMs. 

##### Dynamic Client Server Relationship & Subcontracts

Dynamic client-server relationship because spring kernel was the basis for the network file system. Whether client and server are on the same machine is hidden from the developer. Client and the server can coexists on the same machine, client could be replicated, server could be replicated, and server could be cached. These can be decided at runtime. In subcontract, client stub doesn't know where the server is executing. Communication between client and server domain through the network proxy is done via subcontracts.

##### Java (Oak) Distributed Object Model

- Remote object: accessible from different address spaces
- Remote interface: Declarations for methods in a remote object
- Clients deal with remote method invocation exceptions.
- We can pass remote object references as parameters, but only used value/result mode

##### RMI Implementation - Remote Reference Layer (RRL) => Transport 

- Endpoint (Process): has its own protection domain & table of remote objects 
- RRL does connection management: Setup, teardown, listen; liveness monitoring, choices of transport

##### N-tier Architecture

- Coarse grain session beans - parallelism across clients
- Data Access Object - parallelism within & across client. Business logic is compromised
- Session bean with entity bean (session facade in EJB container) - concurrency, business logic not exposed



# [14 Mar 2023] Distributed Subsystems - Global Memory System

- Often the byproducts of a thought experiment have more lasting impact than the original vision behind it. e.g.yellow-sticky post-it notes but for space exploration, Java for PDA.
- GMS: How can we use peer memory for paging across LAN? 
- Networks are getting fast. Usually we don’t have something in physical memory, we go to the disk. Disk is electromagnetic and it is slow, but as network is getting faster, we can have a local part that is containing the working set of the processes that are running on the local node, and a global part (spare mem) that do work for holding the page load for peer machines. Local and global memory is not static but dynamically shifts depending on workload and memory pressure.
- Changes to the boundary of local and global on each node:

| **Faulting Page X**             | **Faulting node P** | **Node Q with page X** | **Node R with LRU page**              |
| ------------------------------- | ------------------- | ---------------------- | ------------------------------------- |
| In Q’s global                   | L:+1; G:-1          | No change              | No change                             |
| In Q’s global, P’s global empty | No change           | No change              | No change                             |
| On disk                         | L:+1; G:-1          | NA                     | -1 +1 (depending LRU on local/global) |
| Actively shared with Q          | L:+1; G:-1          | No change              | -1 +1 (depending LRU on local/global) |

- Geriatrics: 2 parameters: **epoch**, t (max duration) & the amount of max replacement, **M**

- - During each epoch, all nodes in the cluster sends to the initiator, the age information for the pages. Initiator calculates minimum age of the pages that are expected to be replaced (for each epoch, maximum M pages to be replaced), then we will know the M oldest pages to be evicted. Each node receive {minAge, wi}. The initiator for next epoch is the node with max(wi). 

  - Action at a node on page fault, page y is an eviction candidate:

  - - Age(page y) >= Min Age: discard
    - Age(page y) < MinAge: Send to peer Ni  

  - Global is always a clean copy. Disk writes from subsystems happen unchanged. 

- Implementation in UNIX: Page missing in VM/UBC => Getpages/putpages to GMS. 

- - Access to anonymous pages & FS mapped pages to go through GMS on reads

- Data Structures: VA->UID [IP-Addr | Disk Partition | i-node | offset]

- - UID -> **Page frame directory (PFD)** -> PFN 
  - UID -> **Global Cache Directory (GCD)** -> Ni [dynamic, partitioned hash table]
  - UID -> **Page Ownership Directory (POD)** -> Ni [replicated on all nodes]

- Common case: page non-shared; Node A (POD) & B(GCD) are same node. => quick

- Uncommon case: Miss is possible due to PFD evicted the page and GCD update is in progress / POD stale due to node addition/deletion.



# [21 Mar 2023] Distributed Subsystems - Distributed Shared Memory

- Can we make the cluster work like a shared memory machine?

Sequential Consistency, Eager Release Consistency (RC), and Lazy RC

- Sequential Consistency (SC) => Does not distinguish between accesses to **synchronization** and normal **data** variables. Coherence action on every r/w access. 
- Release Consistency (RC) => Distinguish between data r/w and synchronisation r/w. Coherence action only when **lock released**
- Lazy RC => Between P1 release and P2 acquire, there is opportunity for procrastination. Coherence action at require. 
- Eager (Push mode) vs Lazy (Pull mode) RC: Lazy RC incurs less messages. There is no broadcast during release, only pull when acquire. Lazy RC incurs more latency. 

#### Software DSM

- Fine-grained (individual memory access) sharing in S-DSM is infeasible. Too much overhead for page-level coherence maintenance. 
- Global virtual memory abstraction (Address space partition, address equivalence, distributed ownership)
- During a page fault, DSM got contacted by the OS. DSM software implementation contact owner of page to get the current copy. Ownership for a page is **statically** distributed and known to all the nodes. DSM sends to the manager node of the page. 
- Single writer protocol is easy to implement. We invalidate all copies before write. 
- Page-based S-DSM causes lots of false sharing due to coarse granularity. Data appears shared even though programmatically they are not. A page can consist of many unrelated data structures. 

#### LRC with multi-writer coherence protocols

- P1 has the lock L, and writes to pages X, Y, Z in the critical section. => Calculate Xd, Yd, Zd (diffs). P2 gets the same lock, it has to invalidate copy of X, Y, Z at local acquisition (using LRC), then gets into the critical section. When they want to access the page, DSM fetches and applies the diffs.  
- Implementation: When process tries to write to a page X, the process will make a twin. The original copy is writable. The twin is additional copy of the same page. The thread reaches the release point. The DSM compute the difference between the original version (with modifications) & the twin. The difference computed as runlength encoded diff. All other X is invalidated at the acquisition. After that, we will write protect the original page, and get rid of the twin after the release. 
- If P1 and P3 both writes to X, P2 should apply both diffs from P1 and P2. 
- Apply Xd then Xd’ (in the right order) so that most recent changes is reflected (in case they write to same portion of data).
- If P4 is unrelated to the causality chain of L, it does not have to apply the diffs before read L2. P4 is concurrent with P1, P2 and P3. 
- Multi-writer protocol works only with RC or LRC memory model. With the SC memory model with page granularity, only ONE writer to page at a time. 
- GMS is a system-wide abstraction; DSM is a per process abstraction. GSM is more complicated than DSM as it needs to remember the ages of all pages, & collecting information is non-trivial.

# **[28 Mar 2023] Distributed Subsystems - Distributed File System &** **Internet-scale Computing - GiantScale + Map-reduce**

#### Distributed Subsystems - DFS

- Both **LFS** and **Journalling FS** aims to solve small write problem. JFS **has log files and data files**. LFS has **only log files** and no data files. 
- Traditional NFS does **not** do dynamic management of data and metadata. It is a centralized system in terms of the management of individual files (data and metadata). 
- Distributed File Systems: Metadata management is **dynamically** distributed. Can use cooperative caching of client files. xFS stripe log segments on the disks of the LAN to take advantage of the higher bandwidth that is available in all the distributed nodes. 

Client Reading a File: 

- Own Cache -> file is in client cache, get the data block directly from the client memory.
- Get from Peer Cache -> file is not found on local cache, consult mmap on local memory, reach to the manager to the peer to request for a file. 
- Longest path -> Consult mmap. The remote file manager looks up imap and stripe group map data structure to find the location of inode for the log segments. Go to the storage server to get the index node of the logseg id, then manager looks up the stripe group map to get the storage servers that have the log segment stripes. Stripes of log segments are gotten from the disk, reconstructed the file, and given to the client. 

#### Giant Scale Services

- Giant-scale services are network bound. 

- DQ Principle: Yield: Q = Qc/Q0. Harvest: D = Dv/Df. DQ is a constant, we can either increase Q and decrease D, or vice versa. When we want to increase the capacity of a server, either increase the harvest or the yield. For some failed servers, we can either decrease harvest or yield. 

- **Uptime = (MTBF - MTTR) / MTBF**

- Replication and Partition. When failure occurs, with replication, harvest (D) could remain unchanged, but Q decrease. With partition, Q could remain unchanged, but D decrease.

- Graceful degradation. When server saturation, we can use DQ constant principle.

- - Keep D fixed, and let Q drops. E.g. google search (partial search is OK)
  - Let D decrease, but Q is fixed. E.g. Email (we want to have full harvest)

#### MapReduce

- Several processing steps in giant-scale services are expressible as a MapReduce. 
- Runtime initiates the number of mappers, reducers & data movement.
- In the absence of failures, the number of map instances equals to or **less** than the number of shards of the input. In the absence of failures, the number of reduce instances equals to or **less** than the number of distinct outputs that we want to generate. 
- The number of intermediate files passed from M mappers to R reducers: M*R. 
- Reducer does remote read upon notification from the master that the mapper is done. 
- Master spawns a new mapper since there is no timely response, the original mapper is just slow, it also finishes and notifies the master. Master ignores this mapper. Master spawns a new reducer since there is no timely response: the original reducer is just slow. It finishes and produces a final output file, this will NOT abort the M/R program. 

# **[3 Apr 2023]** **Internet-scale Computing - Coral CDN**

#### Distributed Hash Table

- Placement: Put <key, value>

- Retrieval: Get <key> : get back <value>

- “Put (K, V)” => Places <K, V> at a node N~=K

- Coral CDN => Coral Key-based routing 

- - Greedy leads to tree saturation

  - Routing table: XOR distance from the destination

  - Each hop go to some node that is half the distance to the dst in the node-id namespace Example: From (src)14 to (dst)4, the destination node is 4 (0100) XOR 5 (0101)= 0001 (1). If there is no node at 1, we go to the nearest node. 

  - The table is continuously evolving. 

  - In each hop, we go to the node that is half the distance to dst in node-id namespace. 

  - - 14 XOR 4: 10 => 10/2 = 5 distance… 
    - Target for 1st hop: 14 XOR 4 = 10 => 10/2=5. 
    - There is no node at 5, we go to 4 and RPC for the half distance to node 4. 
    - Next, 0 XOR 4 = 4: 4/2 = 2. 
    - The routing table evolves with the updated entry {4, 5, 7}. 
    - Go to the node with distance = 2 => not found => go to node 5 (closer)

- ##### Coral Sloppy DHT

- - Put (key, value) : value is the node-id of proxy with content for key

  - - Announcing willingness to serve as proxy

  - 2 state variables: 

  - - Full: a particular node store max *l* values for keys 
    - Loaded: max β request for key

- Coral in action [see slides for image]: 

- - Origin server not stressed, meta server is distributed
  - Kishore wants to put <100, 30> to node 100, Use coral based routing and store it
  - Hamed wants to get 100, make bunch of RPC calls, finally reach David node 100, 30 has the content. 
  - Hamed download content from 30, Naomi sends to content. 
  - Hamed serves as a proxy for Naomi
  - Hamed put <100, 60>, David doesn't want to store more than 1 copy for it
  - The middle node becomes the new metadata server for <100, 60>
  - Kenny wants to get(100), and hit the intermediate node <100, 60>
  - Kenny gets the content from Hamed => the origin server is not overloaded



# **[10 Apr 2023]** **Real-Time and Multimedia**

#### Timer latency: 

|                                   | Pro                       | Con                                         |
| --------------------------------- | ------------------------- | ------------------------------------------- |
| Periodic Timer (Polling)          | Periodicity               | Latency for timer event                     |
| One-shot (Precisely set the time) | Timeliness                | Overhead (Fielding the interrupt by the OS) |
| Soft                              | Reduce overhead           | Polling overhead, latency                   |
| Firm                              | Combines all of the above |                                             |

**APIC [One-shot timer]** set by writing a value into a register which is decremented at each memory bus cycle until reaches zero. For example, 100MHz memory bus → 10ns. Fielding interrupts by the OS is the limiting factor.

##### **Firm timer Implementation**

- With a timer-q data structure, sorted by expiry time

- Firm timer combines APIC timer and soft timers

- - APIC timer hardware reprogrammed every few cycles
  - Soft-timers eliminate the need for fielding the one-shot interrupt 

- When APIC timer expires, the scheduler will find the task in timer-q that expires, it will call the interrupt handler if expires. Choose appropriate overshoot distance, we can avoid fielding the interrupts using soft timer.

- Long one shot distance => Possible that several periodic events will go off in the one shot interval. If long one shot timer, user periodic timer and dispatch the one-shot event at the preceeding periodic

##### **Reducing Kernel Preemption latency**

- Explicit insertion of preemption points in kernel

- Allow preemption anytime kernel not manipulating data structures

- Lock-breaking preemptable kernel

- - Split critical sections (acq…rel) => (acq …<manipulate share data> .. release; acq … release). So that after release the shared data, we can preempt the kernel 
  - During the preemption, we can check for expired timers

##### **Reducing Scheduler Latency**

- Proportional Period Scheduling => Q & T adjustable
- Priority-based Scheduling => To prevent priority Inversion, the server executes on behalf of C1 will assume the priority of C1, so that other medium priority threads cannot run

#### **Persistent Temporal Streams**

- False positive: System says something happened which did not
- False negative: System missed to report something that happened
- PTS channel and UNIX sockets: both are internet wide unique abstractions. PTS keeps timestamp metadata with the message content while socket did not
- Persistent Channel Architecture: Live Channel Layer, interaction layer, persistent layer, and backend

# **[18 Apr 2023]** **Failures & Recovery (LRVM and Rio Vista)**

#### LRVM

- Example of OS service that uses persistent data structure: File system

- Log is persisted, while a record is not. The “undo” record is in memory.

- Reads/writes to the persistent state during the transaction is NOT via LRVM

- LRVM synchronously forces the redo logs to the disk at the end_xact => depends

- Traditional DB transactions maintain ACID properties

- - Atomicity: The entire transaction takes place at one or doesn't happen at all
  - Consistency: The database must be consistent before and after the transaction
  - Isolation: Multiple transactions occurs independently without interference
  - Durability: The changes of a successful transaction occurs even if the system failure occurs

- LRVM is per process. If the process is multithreaded, the process has to worry about overlapping regions in set-range calls across threads. LRVM is not responsible for consistency.

- Start from the end of the log for crash recovery to eliminate redundant work

#### Riovista (Rio file cache)

- VM protection to prevent OS errors such as wild writes to file cache during software crash and power failure

- 2 ways of Rio file cache: 

- - File write: Need fsync in the absence of Rio for commitment
  - Mmap file into application space: need msync in the absence of Rio for commitment

- Both not needed for Rio since it is battery backed. Its content will survive power failures

- Upon software crash, file cache data is memory written to disk for recovery

- Upshot: (i) No need for synchronous writes to disk (ii) Write-back of files to disk can be arbitrarily delayed => It helps reduce the amount of I/O to be performed

- If there is an abort => No swat, apply “undo log” to data segment

- Implications of Vista design: (i) No synchronous disk I/O (ii) No redo log (iii) Data segments directly modified during the transactions (iv) Reads/writes to persistent state during the transaction is via Rio file cache

#### Quicksilver (by IBM), 1985

- Transaction tree => for recover from failures
- One quicksilver transaction tree for every sequence of client-server interaction
- Log maintenance at a node in quicksilver is common for all apps that desire recovery management. 
- Quicksilver and LRVM offer recovery management for all applications that desire it.
- Quicksilver is more comprehensive recovery management than LRVM
- A transaction is aborted only when the coordinator initiates an abort.



# **[25 Apr 2023]** **Security**

- Symmetric (private key) encryption uses the same key for encryption and decryption

- Asymmetric (public key) encryption uses different (but related to one another) keys for encryption and decryption. 

- Private key system uses symmetric keys 

- - Data → Enc(data, key) → ciphertext → Dec(ciphertext, key) → Data

- Public Key system => Uses asymmetric keys (pair of keys related mathematically) where the public key published is used for encryption and the private key is used for decryption (one way functions)

- - Data → Enc(data, publickey) → ciphertext → Dec(ciphertext, privatekey) → Data

- With private key encryption, the two parties have to send identity in clear text for every interaction. 

- ### **Andrew File System (AFS)**

- - Chose symmetric because: 

  - - Natural to extend the login facility for students
    - With asymmetric it is difficult to distribute and maintain keys in a campus environment 
    - Symmetric key encryption is more performant

  - Pros and cons: 

  - - Private key system: O(N^2) keys 
    - AFS all users to one server => so only O(N)
    - Unique pair of asymmetric keys for each new user with public key encryption => key distribution problem; more computationally intensive

- Venus-Vice Communication => Bind at the core of communication

- - Authenticate user, authenticate server, and prevent replay attacks

  - RPC session establishment (Bind client-server), Intended client successfully decrypts => Client knows server is genuine (establish server genuine) 

  - “person in the middle” **cannot** decrypt E[Xr+1, Yr] 

  - Server successfully decrypts => Server knows client is genuine (establish client genuine)

  - Person in middle (sniffer) grabs the client response, replays to server => Server will ignore it as a duplicate

  - Client generate new Session key => Don’t want to overexpose HKC

  - Login process: 

  - - Cleartoken: extract handshake key client (hkc)
    - Secrettoken: cleartoken encrypted with key known only to vice => unique for this login session

- If AFS use public key encryption, there is no need for sending user identity in cleartxt. System has to maintain a separate public key for each user. 

- AFS security report card: Mutual suspicion (yes), protection from system for users (no) confinement of resource usage (no - DOS attack) authentication (yes) server integrity (no - physical and social mores)
