# Consistent Hashing

## Why

- Modulo by N approach is used widely to distribute data based on a key identifier
- If the boundaries are fixed this approach works find to partition data across multiple nodes
- The Modulo-N approach fails when N is dynamic as when N changes all keys have to be repartitioned

## How

- Consistent hasing is used in databases like Cassandra to overcome the limitations of Modulo N hashing
- Here is how it works:
    - Say there are 4 nodes. A hash function e.g. MD5, Murmur3 can be used to generate a hash range. Each node is responsible for given range e.g.

      | Node            | Start range	           | End range  | 
      | ------------- |:-------------:| -----:|
      | A	        | -9223372036854775808	    | -4611686018427387904	  |
      | B	        | -4611686018427387903	    | -1	                    |
      | C	        | 0	                        | 4611686018427387903	    |
      | D	        | 4611686018427387904	      | 9223372036854775807	    |
      
     
    - Here is how the following keys will be assigned
      | Partition key  | Hash value | Assigned Node
      | ------------- |:-------------:| -----:
      | johnny	          | -6723372854036780875 | A
      | jim	          | -2245462676723223822 | B
      | suzy	          | 1168604627387940318 | C
      | carol	          | 7723358927203680754 | D
   
  - This is better then the Modulo-N approach as if a node fails then the data that was supposed to be processed by it will be handled by the next in line node
  - This approach can is vulnerable to causing skews because in the case of failures the data doesn't get evenly distributed
  - To handle agains such data skews a concept called virtual nodes is introduced. Virtual nodes (tokens in cassandra), known as Vnodes, distribute data across nodes at a finer granularity. 
  - Vnodes allow each node to own a large number of small partition ranges distributed throughout the cluster. If the token selected is 8 then the node is responsible for 8 smaller ranges of data. For example in the case described above each node can have 8 virtual nodes. Now if the node fails the data that it was responsible for handling gets distributed evenly acroos all other nodes

## What

  - This approach to partition is used in NoSQL databases e.g. cassandra
  - This approach works well to avoid data skews when working with dynamic nodes but is vulenarble to hotspots e.g. in a social media app e.g. instagram an activity caused by a celebrity account can result in too many requests being redirected to a single node. The approach that is taken in such scenarios is to split the hash key further by adding random number to the beginning or end of the key. This would make reads slightly slower as all the associated partitions have to be read. Also it involves an overhead of maintaining a track of the keys that are split
