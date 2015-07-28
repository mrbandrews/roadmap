# bitcoin core roadmap


# preface

This is an effort to document the reference implementation of the Bitcoin protocol, known as "Bitcoin Core."  Bitcoin Core is the C++ code base originally developed by Satoshi in 2009, and now maintained by a small group of programmers.   The code base is open-source and available at GitHub (github.com/bitcoin/bitcoin). 

This covers those parts of the code that initialize the P2P node, connect with peers, exchange blocks and transactions and maintain a blockchain, and handle RPC calls.   I am not covering the GUI, the wallet, or the internal miner.

This document is based on version 0.11.  Any version of Bitcoin Core can be viewed on GitHub so if you want to check out a specific reference go there (select 0.11 from the "tags" dropdown menu.)

This document does not spend a lot of time explaining bitcoin concepts or the bitcoin protocol – that information is available elsewhere and is a prerequisite to studying Bitcoin Core.  If you haven’t already, you should familiarize yourself with the basic structure of blocks and transactions and the main p2p messages. 

The target audience is programmers who have read the Bitcoin whitepaper and probably some other Bitcoin resources (wiki, etc), and now wish to get up to speed on the codebase. 

The purpose is twofold.  First, to help people learn the codebase.  Second, as a matter of principle, this code is important and there should be a document that describes how it works. 

From here on out, I'll refer to the code as "the client." 

Sometimes I'll use the first person in describing the code (e.g., "we send our peer a message…")

## overview of bitcoin core

The client runs on Linux, OSX (Mac), or Windows.   It is written in C++.   It uses boost libraries for various functions, levelDB for the block index and the utxo set, and QT for the gui.  

The client is a multi-threaded application, yet is "largely single threaded" [Hearn]- what is meant is that although there are several threads handling various tasks, most work is done by a single thread: the messaging thread, which constantly communicates with peer nodes, exchanging and validating blocks and transactions. 

The client implements a few layers of abstraction: 
database access (wrappers to LevelDB, _____)  
networking plumbing (managing node connectivity, sockets & streams)
p2p protocol (exchanging & processing messages)
blockchain logic (validating blocks & transactions)
command-line interface & remote access (RPC)


# data storage

There are four key pieces of data storage, all stored in the data directory.  The following descriptions are based on a Stack Exchange post by Pieter Wuille (sipa).

blocks/blk*.dat:   Each of these files stores 128 MB of the full blocks in their raw network-format (meaning the way they arrived on the P2P network).  All other data is redundant in the sense that it can be derived from these data files.    However, to do so on command would be terribly inefficient and that's why we have a block index (metadata about blocks) and a spendable-coins database (utxo's) – see below.  In fact, once we've downloaded these blocks and stored the metadata, we can delete most of these raw files – that's called block pruning and is covered in its own chapter. 

blocks/index/*:  a LevelDB database that stores metadata about blocks.  Without this, finding a block would be intolerably slow. 

chainstate/*:  a LevelDB database containing the UTXO set ("unspent transactions out").   This is useful because the UTXO set perfectly describes the current state of the blockchain.  In a sense, a new block can be viewed as merely being a patch on the chainstate;  it removes some items from the UTXO set and adds others.  Note that utxo's are colloquially referred to as "coins" and so chainstate/* is referred to as "the coins database."   This database was implemented in sipa's "ultraprune" patchset, PR 1673. 

blocks/rev*.dat:  network-format, containing "undo" data necessary for a re-organization.   Undo data are conceptually "reverse patches" to the chainstate. 


# main operations


The client is oriented around several operations.

## a.   Initialization and Startup

Upon startup, the client performs various initialization routines, including starting multiple threads to handle concurrent operations.

b.  Node Discovery and Connectivity

The client uses various techniques find out about other bitcoin nodes that may exist, initiates and maintains connections to other nodes.

c.  Initial Block Download (IBD)

The client downloads the blockchain from the very beginning.  It does so efficiently by first downloading and validating all of the block headers so as to determine the longest chain, and only then downloads all of the block/transaction data.   

d.  Block Exchange

Nodes advertise their inventory of blocks to each other and exchange blocks to build block chains.   Validating a block received from a peer is the essence of the bitcoin consensus system.  A valid block will be added to the node's active chain and relayed to the node's other peers. 

e.  Transaction Exchange

Nodes receive, validate, and relay transactions received from its peers.  The client stores a pool of valid transactions in memory (the "mempool" of transactions.  Remember that word, you will hear it often.)  When the transaction later appears in a newly-mined block, it is removed from the mempool. 

If the user is running the wallet, new transactions created in the wallet are pushed out to peers, and incoming transactions are checked to see if they are payable to the wallet. 

f.  RPC Interface

The client offers a JSON-RPC interface over HTTP over sockets to perform various operational functions and to manage the local wallet.

g.  Block file removal (pruning)

If enabled, the client will remove old, raw block data files.  Block files are only used to help new nodes get up to speed and to handle reorganizations.  Keeping the last 288 blocks on disk allows the client to handle a 2-day reorganization. 

h.  






# 10 key .h/.cpp files

The code is in the src/ directory and a few subdirectories.  Most of the action is in src/.    Let's do a top 10 list – the files that any developer should have a decent grasp of what's in there. 

1) init.cpp

Program startup and initialization.   Reads the command line and gets things organized.   Its steps are well documented in the code, which include loading the block index, loading the wallet, etc. 

2) main.cpp /.h

main.cpp is a behemoth (about 5,000 lines).   Some of what it does: 
- processes messages from other nodes, sends messages to other nodes
- validates & relays transactions
- validates & relays blocks
- maintains the in-memory blockchain and related data structures

main.h contains many of the important constants used by the client and declarations for functions used throughout the program.

4)  coins.cpp/.h

The transaction memory pool. 

5)  net.cpp

Networking classes & routines.  Lots of "plumbing" code that makes the p2p network work. 

6)  util.cpp

Utility functions used throughout bitcoin, eg: 

7) rpc*.cpp  (multiple files)

Remote procedure call handling.   This can be a handy place to hook into the code for debugging purposes:  create an rpc call for yourself, declare the function you want to call in main.h, and then call it on command. 

8) chainparams.h/cpp

Defines main/testnet/regnet, hard-coded seed nodes. 


9) chain.cpp / chain.h



10) txdb.cpp/.h. 




# commit-access devs

The developers who have commit access (meaning their github accounts can merge code changes into the project) are listed below.  It's useful to name them up front as it's sometimes easier to describe some chunks of code (and easier for you, the reader, to go look it up) by referencing the developer who designed or implemented it. 

Wladimir van der Laan (wladimir):  lead maintainer (2014-present)
Gavin Andresen (gavin):  lead maintainer after satoshi (2010-2014)
Pieter Wiulle (sipa):  ultraprune (utxo db), headers-first IBD, lots of other features
Greg Maxwell (gmaxwell): 
Jeff Garzik (garzik): 