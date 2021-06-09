# Replication

## Strong Consistency (Single Copy Systems)

- Primary/Backup or Primary/Replica method
  - Suffers from the risk of inconsistency between primary and replica
- 2PC: Coordinator node and follower nodes
  - Risk of blocking as coordinator node has to wait for the follower nodes
  - Popular among RDBMS 
  - No partition tolerance
- Paxos and Raft are the two partition tolerance algorithms with strong consistency
  - Network partition tolerance means that during network partition only one system is active 
    

## Weak Consistency (Multi Copy Systems)

- Dynamo style databases e.g. cassandra, riak
  - Maintains data as a key\value pair
  - Consistent hashing used by nodes to determine where to read and write data
  - Maintains sloppy quorums for read and writes, ideally (R + W > N)
  - (R + W)> N != Strong Consistency 
    - Nodes are changing in the cluster
    - During network partition the system doesn't behave like one system
    - Dynamo DB is designed as a system that always accepts writes
  - Conflict Resolution is done using Vector clocks, during concurrent updates all versions are sent to the application layer
  - Replica Synchronization
    - Gossip protocol
    - Merkel trees i.e maintain multiple hashes of data e.g. full data hash, half data hash, 25% data hash so on and so forth. During conflict compare hashes to find out divergence   