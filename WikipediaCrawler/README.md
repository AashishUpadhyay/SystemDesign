# Wikipedia Crawler

Wikipedia crawler is an application that is used to crawl as many wiki pedia pages as possible and store their contents in a database

## Requirements

- system should be able to process and store as many wikipedia pages as possible across all langauges
- a page shouldn't get processed twice
- system should be scalable for increased loads
- system should be able to add a new page or update and existing when pages when content changes in wikipedia
- system should be resilient to blacklisting

## Design

### Crawler

A crawler service is a program that process contents of a wikipedia webpage. The processing involves parsing the page to extract all the links embedded withing the page. The crawler also assigns a unique code for the link URL. It stores the unique code and the other metadata information in a database. The blob containing the page contents in stored in a object storeage service e.g. AWS S3.

Once the page gets processed, the crawler should generate a hashcode for the page and add it to a caching service like redis, so that if next time the same link is visited again it knows that it does not have to process the link.

The embedded links that the crawler extracts from a given page are fed to a service broker like rabbitmq. The crawler service reads messages from the service broker.

### Application Logic

- Wikipedia manages pages in different languages that are hosted under different subdomains
- Our crawler services are horizontally scalable such that a group of machines running crawler service can together process webpages hosted under a particulare subdomain
- We can use a system like zookeeper to provision one or more crawlers for one or more langauges
- The crawler services will initially have to be provided some seed links during bootstrapping, there onwards it will use the service broker to manage pages that need to be processed
- Before processing a page the crawler service should check if the hash code for the page is present in the cache or not. If the code is present it should not process the page again
- RSS feeds can be used to add new pages or update existing ones
- To avoid blacklisting our crawlers services should honor the request limits of the the API. Some recommended practices are avoiding too many requests in parallel. If the API supports it should try and read multiple pages at once

### Database

- Our service is write heavy we can choose to store the data in a NoSQL database e.g. cassandra. The data can be partitioned based on language
- Wikipedia used MySQL as a datastore and uses langauges to partition data so may be we can use a relational database as well

## Components

Here are the main components of the application:

- crawlers: stateless microservices used to crawl wikipedia pages
- database: a relational db used to store the metadata for each wiki page processed
- messaging service: a service broker like RabbitMQ used to feed wikipeedia pages to the crawlers
- object storage service: an object storage service like AWS S3 bucket to store wikipedia page content blob
- caching: a cachine service like redis that is used to ensure that a page doesn't get processed twice
