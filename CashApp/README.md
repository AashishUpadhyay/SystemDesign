# Cash App

## Problem Definition:
  - Feature: Allows transfer of cash seamlessly between one another
  - Scale: Scalable to support upto 10 biliion users across the world
  - Conditions: Database Consistency
    - No transaction errors

Questions: Can transfers me made outside the country? (Government rules prohibit this)

## High Level Design

  - The two main components of the application will be
    - api layer
    - database
  - Given the scale that needs to be supported the best strategy is to partition the data across multiple nodes
  - The System can use replication (Single Leader) to support read requests to reduce the traffic on the master node
  - A load balancer can be used to distribute the api traffic evenly across the web servers
  - A routing tier will be needed to route requests to appropriate nodes

## Deep Dive

### Database:
  - We will use a RDBMS database like MySQL
  - For a user we need to maintain two kind of tables
    - User Info
    - User Transactions
      - Id
      - UserId
      - IsDebit
      - FromTransactionId
      - FromUserId
      - ToTransactionId
      - ToUserId
  - Each user and its all transaction information (user data) will be stored in the same shard
  - User data can be partitioned based on the user names (the first letter of the username can be used to partition the data accross 26 shards)


### API:

The APIs that we will probably need are:

- xyz.com\{Id}\transfer
  - POST request
  - Request Body:
    ```
       {
         "to_user_id": "1",
         "amount": "100",
         "currencey": "$"
       }
    ```
- xyz.com\{Id}\balance e.g `xyz.com\999\balance`
  - GET request
- xyz.com\{Id} e.g `xyz.com\999`
  - GET request

### Application

- Transfer (User Sends Money):
  - The application will receive the request to transfer money
  - The payload consists of the from_user and to_user information. This information will be used to determine the node where the relevant data resides
  - The system will initiate the transaction by creating a debit transaction record for from_user. This will be followed by a credit transaction record for to_user.
  - The system uses 2PC using a service like DTC to coordinate transactions between two partitions

- Balance:
  - The balance at any point shows the total amount user has in his account. This is a sum total of all valid transactions
  - If any transaction fails because of node unavailabilty than that transaction is excluded from the balance
  - The transactions are read from the Replica database

- Routing Tier:
  - A routing service allows Application layer to determine the node where the user data resides.
  - The User Data can be partitioned based on user names for e.g. using the first letter of the username will allow us to partition user data across 26 nodes.
  - The partition range and node mapping can be maintained in a coordination service like zookeper
  - TODO: Describe partition strategy


## Bottlenecks and Scale

- Routing Tier can act as a single point of failure
- Distributed transaction can bring down the performance of the system (alternate schemes for consitency)
- Using the partition strategy described above seems a bit of an overengineering when applied from day one 
