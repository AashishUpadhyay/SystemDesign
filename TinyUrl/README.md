# TinyUrl

Tiny Url is an application that supports two primary functions

- Create a short alias(tiny url) for a long URL
- Redirect to the actual long URL when alias (tiny url) is requested

Like any internet application, a tiny url system comprises of the following main components

- database
- api
- business logic
- front end

In this post I have tried to come up with a design for various components of a tiny url system using my knowledge in distributed systems.

## Requirements

Before we begin it is important to assess the scale requirements of our application. A lot of our design choices will depend on the scalability of our Application.

Lets assume that our application needs to support 10 trillion URLs. This is a decent number to start designing out application given there are approximately 1.5 billion websites all over the internet.

If our application is generating 1000 tiny urls per second. Then it would take us more than 300 years to have 10 trillion tiny urls

## Database:

### Entities:

- URL: URL needs to exist as an entity in our database. The primary information that needs to be persisted for a URL are as follows:
  - id
  - code
  - actual_url
  - user_id
  - expiration_date
    The user_id for a url can be null if we allow users to create a tiny url without signinng up in the application. All such urls will be created under free plan. We may decide to purge any url that is created under a free plan in a relatively shorter amount of time
- User: In our application we can allow users to sign-up and buy subscription plans. The subscription plan would allow them to use a given tiny url for an indefinite amount of time
  - id
  - sub_type
  - sub_expiration_date
  - other profile details (e.g. name, address, pincode, payment details etc)

To persist the information we could either choose a RDBMS or a No-SQL database.

A tiny url application does not need relationships between its entities. Most transactions would involve a single object INSERT/UPDATE/DELETE and thus our application can choose to abandon the transactional guarantees of a RDBMS database. Considering all these factors we can choose to select a No-SQL database for our applcation. A No-SQL database scales better and has cost benefits as well

## API:

The APIs that we need to support are:

- xyz.com\makeURL
  - POST request
  - Request Body:
    ```
       {
         "url": "https://google.com",
         "user_id": "1" /*optional*/
       }
    ```
- xyz.com\CODE e.g `xyz.com\SASD5GHSB9`
  - GET request
- xyz.com\CODE e.g `xyz.com\SASD5GHSB9`
  - DELETE request

## Application

The crux of the problem lies in our ability to generate unique Ids for the URLs. Most high level languages support long integers that can be represented using 8 bytes. This allows us to have Ids ranging upto 2^64 in our system, this is way bigger than our requirement i.e. 10 trillion ~ (10 \* 2^40)

All tinyurl systems generate a url with an alphanumeric code e.g. `xyz.com\SASD5GHSB9` (why?...IDK). The Id generated in the system can be converted to a base64 representation if our application is allowed to have alphanumeric code of arbitrary length.

Below are some of the approaches to generate unique Ids for the urls that provide a strong non-collision guarantee.

### Generating Ids on fly

Our Id can be formed the way twitter generates ids for its objects, refer [snowflake](https://github.com/twitter-archive/snowflake/tree/snowflake-2010)

1. time - 41 bits (millisecond precision w/ a custom epoch gives us 69 years)
2. configured machine id - 10 bits - gives us up to 1024 machines
3. sequence number - 12 bits - rolls over every 4096 per machine (with protection to avoid rollover in the same ms)

Advantages:

1. Its very fast because there is no network overhead, the key generation logic is within the application
2. Collison Free, Scalable
3. No additional service or database required for keys
4. Using bits to form an Id also allows us to have tiny url alphanumeric code to be of fixed length, although the total number of URLs that can be supported wiil be only 2^42 now. For example instead of 64 bits we can use 43 bits to genrated a 7 character long alphanumeric code. Here is how the bit allocation can be done for a 7 character code:

   - 31 bits for the timestamp (second precision w/ a custom epoch gives us 69 years)
   - configured machine id - 6 bits - gives us up to have 64 machines
   - sequence number - 5 bits - rolls over every 32 per machine (with protection to avoid rollover in the same second)

Limitations:

Less flexibility w.r.t the nodes where application has to be deployed, if we need more nodes then a change will be required to the bit allocation.

Generating Ids on fly have limitations because a lot of the bits were used by the timestamp or the machine id. If our application can maintain Ids as a sequence number like we were doing for the last 12 bits in the previous approach, we will not have the issues we dicussed earlier.

### Generating Ids using Range-sets

- In this approach our application nodes talk to a Service like Zookeeper to generate Ids
- The Zookeeper acts as a registry for Id rangesets
- When a node receives a request to generate a URL, it request zookeeper to provide it a rangeset
- A rangeset cannot be used more than once
- A node requests for the next rangeset only after it has used all Ids within the current rangeset

![alt text](TinuUrl_KeyGeneration_Zookeeper.png)

The zookeeper does not need to keep all ranges, in the beginning we can start with 1000 ranges each of size 1-million.

If zookeeper is not an option then these ranges can also be maintained in a relational database. Whenever required an application node can get a range from the database and use it to generate a million Ids and then retrieve the next one.

### Generating Ids using Database

The Ids can also be generated using the Identity column in a relational database. All keys can be mainatined in a table. Anytime a request for a key comes, we insert a new row to the table and the generated Id value is returned to the client. This makes our database a a single point of failure. To avoid that we can use two databases to generated unique Ids, one generating odd Ids and the other generating even Ids. This is how Flickr generates uniqueId

Alternatively we can choose to pregenerate Ids and maintain them in a database, whenever a request comes we can user `SELECT FOR UPDATE` to retrive the Id and marking it as used simultaneously.

As an additional improvement the entire key generation logic can be abstracted in a separate service that the application calls over a HTTP request wheenever it needs an Id

## Front End:

Our application needs to provide users an option to provided a long url so that it can be converted to a tiny url. It can be a simple textbox with a submit option.

In addition to this we can have screens for

- sign-up\login
- subscription details
- payment-gateway integration etc

All the other front end features are beyond the scope of this post and therefore will not be discussed more.

## Caching

The application is read heavy and to avoid load on our database the application can use Caching to prevent read requests from going to the database directly

## CDN

To serve customers in different geographical boundaries we can use Content Delivery networks e.g. Amazon cloudfront, fastly etc. This would improve response times and would help in preventing unscrupulous users from carrying out DOS attacks

## Load Balancing

A load balancer can be used to distribute requests to application servers effectively
