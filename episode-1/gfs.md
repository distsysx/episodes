Contains a short summary of GFS

# Major Ideas:
- fault tolerence on commodity hardware
- component failures are norm than exception
- automatic fault detection/recovery must be integral to the system.
- multi-GB files are common 
- most files are updated by appending new data than overwriting existing data.
- once written, files are only read, often sequentially.
- appending becomes the fo-cus of performance optimization and atomicity guarantees, while caching data blocks in the client loses its appeal.
- GFS has weak consistency to get more performance, consistency needs a lot of chit-chat.
- Someone in Github implemented GFS here: git@github.com:merrymercy/goGFS.git
- Google bigtable, mapreduce has been built on top of GFS.

# Most serious limitation of GFS:
- There only one master, and it had to have a table entry for every file and ever chunk, master just ran out of memory when it was used very heavily.
- you can vertically keep scaling a machine forever, that was bottleneck for GFS
- Moreover master had to server too many clients, all these were hard to keep by master, expecially because master had to write changes frequently to disk.
- Many other internal application at google, could not tolerate the consistency problems in case of failed writes.
- Master failover was not automatic, required human intervention.

# Architecture:
- Single master, many chunkservers
- Master only stores meta info
- clients talk to master for meta info , client talk to chunkserver for data operations
- client/chunk-server do not cache any file data to avoid cache coherance issues.
- each chunk size is 64MB
- reads are only big sequential access

# Master Data:
- All in RAM stored as
- Ids also called as handles
- it stores following kvpair
- filename -> array of chunk Ids(NV)
- chunk ID/handle -> {list of chunk servers(V), version(NV), primary(V), lease expiration(V)}
- some these data (*NV) are flushed to disk as Log and checkpoints on disk
- there exists 3 replicas (-> 1 primary and 2 secondaries) per chunk by default 

# Reads:
- name of file, offset -> Master
- Master --> back chunk ID, List of servers. Client caches result
- client --> chunk server 
- chunk server (data) --> client

# Writes:
- client -> append(filename, []byte) --> Master
- Master --> chunk server details (which knows where is the last chunk )
- Client needs to know = location of last chunk, because onto which we can APPEND more data
- reading can be done with any chunk server, but writing should be performed only on the primary chunk server
- If primary already exist:
    - client request primary for writes (appending)
    - client transfer the data to primary and all secondary chunk servers and they store this new data in a temp location
    - then p,s will conform that data came suceessfully as an ACK, then client will ask p,s to append it to disk permanently
    - primary instructs the secondaries to write the data to disk at the end of the file(using offset), and once secondary say "YES" that they wrote to disk, then primary will reply success to client (to conform the write action) else primary replies "NO" to client. Client is expected to start the process from the beginning once again if it gets "NO"
    - Incomplete writes can happen, i.e, client asks the data to append, some replicas write new data, some didn't write, leaving an inconsistent state. ( expected behaviour)
    - Client usually re-issues write to fix the above inconsistency. so eventually all the replicas and primary will contain the latest data from client
- If No primary ?
    - If there is no primary, master needs to know which chunk server has upto date copy of the chunk 
    - There can be a possibility that there is no primary, that master needs to handle
    - Master uses version number to find out the UP-TO date chunk server and that version number is non-volatile
    - Master assigns these version numbers to the chunk server and everyone remembers i t
    - For each chunk master decides the primary and secondary and assigns a new version number
    - master and all chunk servers stores the version info to disk
- Master leases primary for a specific time, after that a new primary will be assigned

# Split brain problem: (presence of >1 primary):
- primary should be always one
- it is usually caused by network partition
- master in GFS solves this problem this way: Master gives a primary a lease / TTL, after that it ceases to be primary
- both master and primary remembers this lease TTL. once it expires , primary will cease to be primary and stops further write request from clients. so master designates a new primary
- Client comes with write request -> master tries to talk to primary -> primary failed -> master checks lease TTL and asks client to wait until the lease expires, only then master will go about finding the next suitable candidate for primary. If master doesn't wait and assign a new primary , and if the old primary recovers from its failure, then we we will have two primaries alive and that will break the system.

# How to upgrade GFS to achieve strong consistency ? Two-Phase commit.
- In short => Strong consistency can be achieved by adding => More chit-chats between the nodes.
- Ability to track track and handle retry request to handle the inconsistent write failure that described in write passage.
- If primary tells something to secondary -> secondary does it somehow and doesn't return error, in case secondary has a non-recoverable error, then primary has to somehow kickout this secondary and assign a new secondary
- If primary asks a new data to write to secondary, secondaries should be careful not to expose the new data to the client readers. and moreoever write has to be performed in some phases. eg phase-1: primary asking secondaries -> "Can you guys do this write ?". If primary gets an "YES" it can execute phase-2: to ask all of them to write the data. This is called two-phase commit.
- If primary suddenly crashes, a new secondary might be promoted to primary. but this transfer has to happen gracefully such that, the new primary will make sure to resolve if there is any incomplete transaction by previous master.

# What happens of master crashes ?
- Master periodiacally stores the state as Checkpoints, if master is crashed, it will be started from the most recent checkpoint

# What to learn ?
- What is linux vnode ?
- Linux buffer cache ?
- Typical file system block size ?
- Write a go program to hold persistant TCP connection for a long time ?
- What is lazy space allocation ?