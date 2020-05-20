# TinyUrl
Tiny Url is an application that supports two primary functions
  * Create a short alias(tiny url) for a long URL
  * Redirect to the actual long URL when alias (tiny url) is requested
   
Like any internet application, a tiny url system comprises of the following main components 
* database
* api
* business 
* front end

In this post I have tried to come up with a design for various components of a tiny url system using my knowledge in distributed systems.

## Requirements
Before we begin it is important to assess the scale requirement of our application. A lot of our design choices will depend on the scalability of our Application.

Lets assume that our application needs to support 10 trillion URLs. This is a decent number to start designing out application given there are approximately 1.5 billion websites all over the internet.

If our application is generating 1000 tiny urls per second. Then it would take us more than 300 years to have 10 trillion tiny urls

## Database:

### Entities:

  * URL: URL needs to exist as an entity in our database. The primary information that needs to be persisted for a URL are as follows:
    - id
    - code
    - actual_url
    - user_id
    - expiration_date
    The user_id for a url can be null if we allow users to create a tiny url without signinng up in the application. All such urls will     be created under free plan. We may decide to purge any url that is created under a free plan in a relatively shorter amount of time
  * User: In our application we can allow users to sign-up and buy subscription plans. The subscription plan would allow them to use a       given tiny url for an indefinite amount of time
    - id
    - sub_type
    - sub_expiration_date
    - other profile details (e.g. name, address, pincode, payment details etc)
    
To persist the information we could either choose a RDBMS or a No-SQL database. 

A tiny url application does not need relationships between its entities. Most transactions would involve a single object INSERT/UPDATE/DELETE and thus our application can choose to abandon the transactional guarantees of a RDBMS database. Considering all these factors we can choose to select a No-SQL database for our applcation. A No-SQL database scales better and has cost benefits as well    

## API:

The APIs that we need to support are:
* xyz.com\makeURL
   - POST request
   - Request Body: 
     ```
        {
          "url": "https://google.com",
          "user_id": "1" /*optional*/
        }
     ```
   
 * xyz.com\{CODE} e.g  `xyz.com\SASD5GHSB9` 
   - GET request
   
 * xyz.com\{CODE} e.g  `xyz.com\SASD5GHSB9` 
   - DELETE request

## Application

Our application needs to support 10 trillion URLs which means we need a way to generate 10 trillion unique codes. We can choose base64 encoding to generate unique codes for our URL. An eight character code using base64 encoding would allow us to have 256 trillion (64^8) unique codes.

## Front End:
