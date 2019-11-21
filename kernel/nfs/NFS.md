# Network File System

 Network File system (NFS) is a distributed file system protocol originally developed by Sun Microsystems(Sun) in 1984, allowing a user on a client computer to access files over a computer network much like local storage is accessed. NFS, like many other protocols, builds on the [Open Network Computing Remote Procedure Call](https://en.wikipedia.org/wiki/Open_Network_Computing_Remote_Procedure_Call) (ONC RPC) system. The NFS is an open standard defined in a [Request for Comments](https://en.wikipedia.org/wiki/Request_for_Comments) (RFC), allowing anyone to implement the protocol.

## NFSv2

Version 2 of the protocol [RFC1094](https://tools.ietf.org/html/rfc1094) originally operated only over **User Datagram Protocol (UDP)**. It designers meant to keep the server **side stateless**, with **locking implemented outside of the core protocol**.

## NFSv3

Version 3 [RFC1813](https://tools.ietf.org/html/rfc1813):

* support for 64-bit file sizes and offsets, to handle files larger than 2G.
* support for asynchronous writes on the server, to improve write performance.
* additional file attributes in many replies, to avoid the need to re-fetch them.
* a READDIRPLUS operation, to get file handles and attributes along with file names when scanning a directory.
* assorted other improvements.

## NFSv4

* Version 4 ([RFC3010](https://tools.ietf.org/html/rfc3010), [RFC3530](https://tools.ietf.org/html/rfc3530), [RFC7530](https://tools.ietf.org/html/rfc7530)), influenced by **AFS** and **CIFS**, includes performance improvements, mandates strong security. and introduces a **stateful** protocal.
* Version 4.1 ([RFC5661](https://tools.ietf.org/html/rfc5661)) (2010) aims to provide protocol support to take advantage of clustered server deployments including the ability to provides scalable parallel access to files distributed among multiple servers (pNFS). 
* Version 4.2 ([RFC7862](https://tools.ietf.org/html/rfc7862)) (2016) was published with new features including: server-side clone and copy, application I/O advise, sparse files, space reservation, application data block, labeled NFS with sec_label accommodates any MAC security system, and two new operations for pNFS (LAYOUTERROR and LAYOUTSTATS).

## XDR (External Data Representation)

**External Data Representation** (**XDR**) [RFC1014](https://tools.ietf.org/html/rfc1014) is a [standard](https://en.wikipedia.org/wiki/Technical_standard) [data serialization](https://en.wikipedia.org/wiki/Data_serialization) format, for uses such as [computer network](https://en.wikipedia.org/wiki/Computer_network) protocols. It allows data to be transferred between different kinds of computer systems. Converting from the local representation to XDR is called *encoding*. Converting from XDR to the local representation is called *decoding*. XDR is implemented as a software library of functions which is portable between different [operating systems](https://en.wikipedia.org/wiki/Operating_system) and is also independent of the [transport layer](https://en.wikipedia.org/wiki/Transport_layer).

XDR defines the size, byte order and alignment of basic data types such as string, integer, union, boolean and array. Complex structures can be built from the basic XDR data types. Using XDR not only makes protocols machine and language independent, it also makes them easy to define.

XDR uses a base unit of 4 bytes, serialized in [big-endian](https://en.wikipedia.org/wiki/Big-endian) order; smaller data types still occupy four bytes each after encoding. Variable-length types such as string and opaque are padded to a total divisible by four bytes.

## RPC

Remote procedure calls [RFC1057](https://tools.ietf.org/html/rfc1057) are synchronous, that is, the client application blocks until the server has completed the call and returned the results.

### NFS Advantage

* sharing
* centralized administration
* security

### Workflow

![1571813397173](/home/wrsadmin/.config/Typora/typora-user-images/1571813397173.png)

### Implementation of NFSv2

NFSv2 was the standard for many years; small changes were made in moving to NFSv3, and larger-scale protocol changes were made in moving to NFSv4.

In NFSv2, the main goal in the design of the protocol was **simple and fast server crash recovery**. In a multiple-client, single-server environment, this goal makes a great deal of sense; any minute that the server is down (or unavailable) makes all the client machines (and their users) *unhappy and unproductive*. Thus, as the server goes, so goes the entire system.

#### Key to Fast Crash Recovery : statelessness

The parameters to each procedure call contain all of the information necessary to complete the call, and the server does not keep track of any past requests.

All of the procedures in the NFS protocol are assumed to be  synchronous.  When a procedure returns to the client, the client can assume that the operation has completed and any data associated with the request is now on stable storage.  For example, a client WRITE request may cause the server to update data blocks, filesystem information blocks (such as indirect blocks), and file attribute information (size and modify times).  When the WRITE returns to the client, it can assume that the write is safe, even in case of a server crash, and it can discard the data written.  This is a very important part of the statelessness of the server.  If the server waited to flush data from remote requests, the client would have to save those requests so that it could resend them in case of a server crash.

#### NFSv2 protocol

* file handle: used to uniquely describe the file or directory a particular operation is going to operate upon.

  Think of a file handle as having three important components: a volume identifier, an inode number, and a generation number; together, these three items comprise a unique identifier for a file or directory that a client wishes to access.

  1. Volume identifier informs the server which file system the request refers to (an NFS server can export more than one file system);

  2. Inode number tells the server which file within that partition the request is accessing.

  3. Generation number is needed when reusing an inode number.

     NFSPROC_GETATTR	file handle; returns: attributes
     NFSPROC_SETATTR	 file handle, attributes; returns: –
     NFSPROC_LOOKUP	 directory file handle, name of file/dir to look up; returns: file handle
     NFSPROC_READ		  file handle, offset, count; data, attributes
     NFSPROC_WRITE		file handle, offset, count, data; attributes
     NFSPROC_CREATE	  directory file handle, name of file, attributes; –
     NFSPROC_REMOVE	directory file handle, name of file to be removed; -
     NFSPROC_MKDIR		directory file handle, name of directory, attributes; file handle
     NFSPROC_RMDIR		directory file handle, name of directory to be removed; -
     NFSPROC_READDIR	file handle, count of bytes to read, cookie; returns: directory entries, cookie (to get more entries)

* Attributes : just the metadata that the file system tracks about each file, including fields such as file creation time, last modification time, size, ownership and permissions information, and so forth, i.e., the same type of information that you would get back if you called stat() on a file.

  ![1571817539698](/home/wrsadmin/.config/Typora/typora-user-images/1571817539698.png)

![1571817574239](/home/wrsadmin/.config/Typora/typora-user-images/1571817574239.png)

#### Handling Server Failure With Idempotent Operations

When a client sends a message to the server, it sometimes does not receive a reply.

The ability of the client to simply retry the request (regardless of what caused the failure) is due to an important property of most NFS requests: they are **idempotent(unchanged)**.

The heart of the design of crash recovery in NFS is the **idempotency** of most common operations.

A small aside: some operations are hard to make idempotent. I.E. mkdir.

Clients and servers may need to keep caches of recent operations to help avoid problems with non-idempotent operations. For example, if the transport protocol drops the response for a Remove File operation, upon retransmission the server may return an error code of NFSERR_NOENT instead of NFS_OK. But if the server keeps around the last operation requested and its result, it could return the proper success code. Of course, the server could be crashed and rebooted between retransmissions, but a small cache (even a single entry) would solve most problems.

**Life is not perfect!**

#### Improving Performance : Client-side Caching

The NFS client-side file system caches file data (and metadata) that it has read from the server in client memory. Thus, while the first access is expensive (i.e., it requires network communication), subsequent accesses are serviced quite quickly out of client memory.

The cache also serves as a temporary buffer for writes. When a client application first writes to a file, the client buffers the data in client memory (in the same cache as the data it read from the file server) before writing the data out to the server. Such write buffering is useful because it decouples application write() latency from actual write performance, i.e., the application’s call to write() succeeds immediately (and just puts the data in the client-side file system’s cache); only later does the data get written out to the file server.

##### The Cache Consistency Problem

Two problems:

* update visibility
* stale cache

Resolved:

* flush-on-close or close-on-open
* NFSv2 clients first check to see whether a file has changed before using its cached contents. Specifically, before using a cached block, the client-side file system will issue a **GETATTR request** to the server to fetch the file’s attributes. 
  * When the original team at Sun implemented this solution to the stale cache problem, they realized a new problem; suddenly, the NFS server was flooded with GETATTR requests. An **attribute cache** was added to each client. A client would still validate a file before accessing it, but most often would just look in the attribute cache to fetch the attributes. The attributes for a particular file were placed in the cache when the file was first accessed, and then would timeout after a certain amount of time (3 seconds).  Thus, during those three seconds, all file accesses would determine that it was OK to use the cached file and thus do so with no network communication with the server.

![read/write/sync](/home/wrsadmin/.config/Typora/typora-user-images/1571813974410.png)

###### Server-Side Write Buffering

NFS servers must commit writes to persistent media before returning success; otherwise, data loss can arise.

### NFSv3 implementation

* Solving the write throughput bottleneck
* Minimizing the work needed to create an NFS Version 3 implementation given an existing NFS
  Version 2 implementation 
* Ensuring that implementation of the new protocol is feasible on less-capable client operating systems
* Completely documenting the resulting protocol and annotating it with implementation examples to aid developers
* Deferring new features to subsequent revisions of NFS due to time constraints

#### Major Changes:

* Sizes and offsets are widened from 32 bits to 64 bits.
* The WRITE and COMMIT procedures allow reliable asynchronous writes.
* A new ACCESS procedure fixes known problems with super-user permission mapping and allows servers to return file access permission errors to the client at file open time to provide better support for systems with Access Control Lists (ACLs)
* All operations now return attributes to reduce the number of subsequent GETATTR procedure calls
* The 8KB data size limitation on the READ and WRITE procedures is relaxed
* A new READDIRPLUS procedure returns both file handle and attributes to eliminate LOOKUP calls
  when scanning a directory
* File handles are of variable length, up to 64 bytes, as needed by some implementations
* Exclusive CREATE requests are supported
* File names and path names are now specified as strings of variable length, with the maximum length negotiated between the client and server
* The errors the server can return are enumerated in the specification—no others are allowed
* The notion of blocks is discarded in favor of bytes
* The new NFS3ERR_JUKEBOX error informs clients that a file is currently off-line and that they should try again later

The NFS Version 3 protocol **requires that modified data on the server be flushed to stable storage before replying**. Only **asynchronous writes are excepted**. 
NFS clients **block on close(2) until all data is flushed to stable storage on the server**, to return any errors to the application that might occur during delayed writes (for example, out of space).

NFS clients are decidedly **not stateless**. NFS clients hold modified data that has not been flushed to the server as well as cache file handles and attributes. Clients typically use attribute information, such as file
modification time, to validate cached information. When a client crashes no recovery is necessary for either the client or the server.

NFS Version 3 **asynchronous writes** eliminate the synchronous write bottleneck in NFS Version 2. When a
server receives an asynchronous WRITE request, it is permitted to reply to the client immediately. Later, the
client sends a **COMMIT request to verify that the data has reached stable storage**; the server must not reply to the COMMIT until it safely stores the data.

![1571826373112](/home/wrsadmin/.config/Typora/typora-user-images/1571826373112.png)
