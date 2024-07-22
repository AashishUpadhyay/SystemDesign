# Overview

- Robinhood is a brokerage that allows its users to perform trades in stock exchange like NASDAQ, S&P500 etc
- This is an example where we use relational database, single leader replication and partitioning to scale.

# Functional Requirements

- A brokerage typically allows the following:

  - Users can register themselves in the brokerage and get an account created for themselves. A user then uses a username\password to login to the brokerage to use its services.
  - Users van view stocks listed in an exchange
  - Users can place orders (buy/sell) using the brokerage. These orders could be of two types Market Order or Limit Order
  - Movement of money between users bank and the brokerage
  - The brokerage shows how a stock has performed over time(1w, 1m, 6m, 1yr, 3yr, 5yr, all). It also shows information like MarketCap, trade volume, 52w highest price, 52w lowest price, P/E ratio etc
  - Users can add stocks to their watchlist, normally this is a fixed set of stocks that you can pick and the brokerage will notify you for changes that you may want to track about the price movement of those stocks
  - Users can view the size of their portfolio. The transactions they had performed such as (buy, sell, dividends etc)
  - Financial reports such as capital gains and IT related forms can be downloaded

# Nonfunctional Requirements

- Highly Available. A typical brokerage running in a country is used by residents of that country. The number of users is high and its expected that the system be highly available.
  - Availabilty is improved by ensuring that two main components of the application work well under high traffic volume. These two components are web servers and database.
  - To improve availability of web servers tools like load balancers, rate limiting, CDNs can be used
  - Improving availability of databases involve implementing right strategies for replication and partitioning. An important factor that helps in deciding the strategy is the scale that our application needs to support. To assess scale it is important to understand a few metrics such as:
    - dailiy active users
    - number of trades per user
- Consistency: Different operations supported by a brokerage can have different consistency guarantees. For example:
  - Profile Updates, Order Submissions, Order Executions have to be strongly consistent
  - Portfolio Updates, Watchlist can be eventually consistent
- Security: Authentication and Authorization
- Low latency: A brokerage should be highly performing. But different functions of a brokerage can have different latencies for example:
  - Data that brokerage maintains internally in its own database can be made available to the user in a second.
  - Order submission and Order execution is dependant on exchanges. The latency should be low but it may get affected by errors in the network or on exchange side. These are normally covered in the Terms and Conditions
  - The prices displayed in the brokerage also are delayed. In the design we will assume that this delay is less than a second.

# Calculations

- Assuming our brokerage supports 10 million DAU, and each user is expected to perform 5 trades in a day.
  - Total number of write requests 50 million in 7 hours. Around 2000 requests per second
  - Our application servers should be able to serve more than 2000 requests per second
- The read operation that our users may be performing can be of the following types:
  - View Portfolio
  - View Trades
  - Download Reports
  - View Watchlist
  - Search stocks
- View Portfolio, View Trades, Report downloads can be performed asynchronously. Portfolio update does not have to display the actual value. A users portfolio calculations can be handled by batch jobs that run at scheduled frequencies
- Two of the read features that may consume resources are Watchlist and Search
  - The brokerage needs to maintain time series data for each stock listed in the brokerage. This will be used to display the user current price of a stock. The price of the stock in a given day needs to be updated periodically.
  - The stock price updates executed in a day will also lead to updating a users watchlist and sending notifications for alerts that the user has configured in the system

# High Level Design

- At a high level we may need the following services:

  - User Management: A service to manage user information
  - Order Management: A service that manages orders a user places for a given duration
  - Transaction Management: A service that manages the transactions of for a user
  - Stocks Management: A service that manages the information about stocks. We will maintain information on each tick
  - Notifications: A service responsible for updating watchlists and sending notifications
  - Reporting: A service responsible for generating reports

## Service Design

- User Management: The service is responsible for managing user information. The database needs to maintain following fields:
  - User Info: user_id, user_name, password, email, phone, portfolio, created_date
  - User Bank Info: bank_id, user_id
  - Watchlists: A list of stocks user wants to track
  - The service is also responsible for authentication and authorization
- Order Management: A service responsible for managing orders. The database needs to maintain following fields:
  - Order Info: order_id, buy/sell, ticker, order_type (limit/market), order_date, order_fill_status (complete, incomplete, partial_complete), expiry_date
  - Order Details Info: order_detail_id, order_id, price, quantity, transaction_id
- Order Processing: Order processing service is a horizontally scalable service responsible for managing lifecycle of an order. The service has following responsibilities:
  - Submit orders created by user to the exchange so that they can be executed
  - Update the Order Management service once an order is executed
- Transaction Management: Manages transactions. The database needs to maintain following fields:

  - Transaction Info: txn_id, txn_type (orderfill, bank_movement, dividends)
  - Transaction Detail Info: txn_detail_id, txn_id, amount, is_debit, party_id, party_type (user, bank, exchange)

    - User moves money from bank account
      txn_id, bank_movement
      txn_detail_id, txn_id, amount, credit entry, user id, user

    - Buy Order fulfilled
      txn_id, orderfill
      txn_detail_id, txn_id, amount, debit entry, user id, user

    - Sell Order fulfilled
      txn_id, orderfill
      txn_detail_id, txn_id, amount, credit entry, user id, user

    - Dividends Received
      txn_id, dividends
      txn_detail_id, txn_id, amount, credit entry, user id, user

- Stocks Management:
  - Manages stock data. A search index that maintains time series data
  - We also need to manage a list of stocks that have been added into the watchlists. A different index
  - The feed received from the Exchange needs to be analysed and if the feed contains tickers that are in watchlists, we need to update the watchlists and evalaute if notifications have to be sent
- Notifications Service: Responsible for sending notifications to the user.
- Reporting Service: This service runs as a batch processing unit and is reponsible for the following:
  - User portfolio Calculation
  - Report generation for the user for tax reporting, capital gains, trade information etc

# Low Level Design

## User Management Service:

- User Management Service is responsible for manging user information
- The user information is persisted in a RDBMS like postgres
- Given the number of users we need to support is 10 million DAU and 100 million total users. A postgres can be run in Primary/Secondary configuration (1 Primary Database and 2 Secondary databases)
- The writes will always be performed in the Primary database and the reads will be from Secondary database.
- User Management Service will also issue tokens that will be used to execute operations in other services such as order submission, report generation, view transactions etc
- Users information such as Portfolio Update and watch lists will be maintained in a Cache.
- Watchlists will be fetched from the Search Index for the stocks
- It supports the following APIs:
  - Add user POST `usermanagement.api.com\user`
  - Update portfolio POST `usermanagement.api.com\user\portfolio`
  - Update watchlist POST `usermanagement.api.com\user\watchlist`

## Order Management:

- Order Management Service is reponsible for executing orders
- The number of orders expected in a day are 50 million. This information can be maintained in a postgres database. The orders table can be partitioned on a daily basis so that for each day we have a fresh table to start order management
- In order to improve performance we can maintain orders inmemory using the data structures described below:

  ```
  class LimitOrderBook {
    LimitOrderNode buyTree;
    LimitOrderNode sellTree;
    dict buyOrders;
    dict sellOrders;
  }

  class LimitOrderNode {
    string ticker;
    float price;
    OrderNode orders;
    LimitOrderNode left;
    LimitOrderNode right;
    LimitOrderNode parent;
  }

  class MarketOrderBook {
    OrderTree buy;
    OrderTree sell;
    dict buyOrders;
    dict sellOrders;
  }

  class OrderNode {
    string ticker;
    float price;
    float quantity;
    OrderNode prev;
    OrderNode next;
  }
  ```

  - The limit orders are stored in a in-memory datastructure called LimitOrderBook that internally maintains two trees for buy and sell orders
  - Adding a limit order to LimitOrderBook is performced in O(LogM) where M is the number of orders or a type (buy\sell)
    - The buy tree maintains the limit orders in the decreasing order of prices
    - The sell tree maintains the limit orders in increasing order of prices
  - A LimitOrderNode maintains a list of orders at a given price. OrderNode is a linked list and orders are maintained based on the time they are created
  - The dictionaries buyOrders and sellOrders in the LimitOrderBook and MarketOrderBook allow executing\cancelling of orders in O(1) time
  - The market orders are maintains in MarketOrderBook as a linked list
  - The in-memory datastructure needs to handle concurrency as orders can be added, executed or cancelled concurrently
  - Order management service will push orders asynchronously to Kafka topic. The order processing service will process the orders submitted by order management service
  - It supports the following APIs:
    - Add order POST `ordermanagement.api.com\order`
    - Complete order POST `ordermanagement.api.com\order\complete`
    - Cancel order watchlist POST `ordermanagement.api.com\order\cancel`

### Problems:

- How do we add scale horizontally?
  - How multiple nodes be handled when the data is maintained in-memory?
  - How data co-ordination happens between in-memory data strcuture and the underlying database?
- Problem Description: I have a service that maintains an inmemory data strcuture or orders. To scale horizontally more instance of service can be added and the orders will be divided between those instances based on order id using hashing. Hash function would be like (order_id/no of nodes). What options can be employed to handle node failures?
  - Consistent Hashing

## Order Processing Service:

- The service is a consumer of records from Kafka topics
- It listens to two Kafka topics
  - Orders to process: Records submitted to the topic are submitted to the excchange for execution
  - Orders processed: Records submitted to the topic are updated in the order management service for marking them as complete or cancellation
- During peak hours more nodes can be added to handle the increased load in the number of orders to be processed

## Stock Management Service:

- Stock management service maintains the data feeds received from exchange
- The service maintains the data in a search index
- The index is a time series data
- More instances of the service can be added to handle additional scale

## Stocks alerats

- A service that maintains a list of ticks and events registered
- Watches feeds from the exchange and evaluate events on those feeds and sends notifications. Critical stock movements are prioritiseed.

## Notification Service:

- Notification service sends notification to users when a stock alert is generated.
- Exponential back-off when system is unable to send notification

## Reporting Service

- A service that generates reports for the user
