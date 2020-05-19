# TinyUrl
Tiny Url is an application that supports two primary functions
  * Create a short alias(tiny url) for a long URL
  * Redirect to the actual long URL when alias (tiny url) is requested
   
Like any internet application, a tiny url system comprises of various components e.g. database, api, application logic and front end. And this is my attempt to design various components of the system using my knowledge in distributed systems.

Database:

Entities:

  * URL: The primary information that needs to be persisted with the URL entity is as follows:
    - id
    - code
    - actual_url
    - user_id
    

API:

Application:

Front End:
