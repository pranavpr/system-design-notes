---
layout: post
title: Netflix System Design
date: 2020-04-26 21:05 +0530
categories:
  - System Design
image: /assets/images/netflix-system-design-cover.jpg
---

Netflix is one the largest on-demand video streaming service over the internet. Netflix allows users to stream and watch videos which are available on its platform. Let try to design a system similar to Netflix.

In this design, we would focus on performance, scalability, security and resiliency. So without further ado, let's get started.

<!-- prettier-ignore -->
* TOC
{:toc}

### Requirements

Let's look in to some of the functional and non-functional requirements before we start to design the system.

#### Functional Requirements

1. Users should be able to create and account and subscribe for a plan.
2. Users should be able to manage multiple profiles.
3. Users should be able to search for a video title.
4. Users should be able to watch a video.
5. Netflix employees should be able to upload a video from backend to make it available on Netflix platform.

#### Non-functional requirements

1. System should be highly reliable. Any video uploaded should not be lost.
2. System should be highly available.
3. Users should be able to stream videos in realtime without any lag.

#### Out of scope

1. Video recommendations
2. Most popular videos
3. Subtitles
4. Grouping of videos (e.g. TV Series, treat each video as independent)

### Capacity Estimation

Let's do some back-of-the-envelope calculations to estimate the bandwidth and storage required.

#### Assumptions

1. Total number of daily active users = 100 million
2. Average number of videos watched by each user per day = 5
3. Average size of one video = 500 MB
4. Average number of video uploaded per day from backend = 1,000

#### Bandwidth Estimations

Total number of videos watched per day = 100 million \* 5 = 500 million  
Total egress per day = 500 million \* 500 MB = 250 PB (Peta Byte)  
Egress bandwidth = 29.1 GB/sec

Total ingress per day = 1,000 \* 500 MB = 500 GB  
Ingress bandwidth = 5.8 MB/sec

#### Storage Estimations

Total storage required in 5 years = 500 GB \* 5 \* 365 = 912.5 TB

Note that Netflix creates multiple formats and resolutions for each video optimized for different device types. Above storage estimation ignores this factor and hence this number can change based on number of variants Netflix stores for each video.

### Detailed Component Design

As evident from calculations above, the system is read heavy. Hence our focus should be on building a system which can retrieve videos quickly.

{% include image.html image="netflix-system-design.png" alt="Netflix System Design" %}

Let's look in to each service in detail.

#### Users Service

Users Service would be mainly responsible for user authentication and profiles. This service would persist the data in a relational database like MySQL or PostgreSQL. We need strong [ACID](https://en.wikipedia.org/wiki/ACID){:target="\_blank"} properties for the set of data we have and hence RDBMS is a suitable choice.

#### Subscriptions Service

Subscriptions Service would be used to manage the subscription of the users. Since data processed by this service are highly transactional in nature, RDBMS makes a suitable choice.

#### Videos Service

Videos Service would be responsible for surfacing videos to end users. This service will store videos metadata in a RDBMS like MySQL or PostgreSQL. However for quick response time, this service would implement a write-around cache. This can be implemented using a in-memory cache like Redis or Memcached.

#### Transcoder Service

Transcoder Service is one of the key backend service. This service would be responsible for following:

- Checking the quality of uploaded videos (frame drops etc.)
- Compressing the video with different codecs
- Creating different resolutions of the video

Once a video is uploaded to Transcoder Service from backend, it will upload the same to a internal distributed storage like a S3 bucket and add entry to database.

Post that it would publish a message in queue which could be implemented using Kafka or RabbitMQ. One other side, workers would consume messages from queue, download the video from internal S3 bucket and transcode it to different formats. Once transcoding is complete, worker would upload the video to external S3 bucket and update the status of video in database as active for viewing by end users.

The worker would also add an entry of the video metadata in search database which supports full text search. This would enable the end users to search for videos using their title or summary.

Finally the video from external S3 bucket would also be cached over a [CDN](https://en.wikipedia.org/wiki/Content_delivery_network){:target="\_blank"} for reducing latency improving playback performance.

#### Search Service

Search Service would enable end users to search for a video using metadata like title, summary etc. This service would be backed by a database supporting full text search. Elasticsearch or Solr can be used for for the same as both these support full text searching. This service can also rank the results based on recency and popularity for better user experience.

### Database Schema

The database schema containing most important tables is illustrated below:

{% include image.html image="netflix-database-schema.png" alt="Netflix Database Schema" %}

Database schema is pretty straight forward and self explanatory.

### APIs

Different services would expose [REST APIs](https://en.wikipedia.org/wiki/Representational_state_transfer){:target="\_blank"} to interact with clients or other services. Below are some major APIs:

#### User Sign-up

This API would be used while user signs-up on the platform.

Request:

```
POST /api/v1/users
X-API-Key: api_key
{
  name:
  email:
  password:
}
```

We would use `POST` HTTP method for this API as we are creating a resource. `X-API-Key`, which is passed in HTTP header, is the API Key to identify different clients and do rate limiting.

Response:

```
201 Created
{
  message:
}
```

HTTP response code `201` tells that user has signed-up successfully. Other possible HTTP codes for failure scenarios are below:

```
400 Bad Request
409 Conflict
500 Internal Server Error
```

#### User Sign-in

Request:

```
POST /api/v1/users/session
X-API-Key: api_key
{
  email:
  password:
}
```

We would use `POST` HTTP method for signing up as we are creating a new session.

Response:

```
200 OK
{
  auth_token:
}
```

On successful sign-in, API should return an `auth_token` which can be passed in header for successive API calls which require authentication. `auth_token` can be generated using [JWT](https://jwt.io/introduction/){:target='\_blank'}.

#### User Sign-out

Request:

```
DELETE /api/v1/users/session
X-API-Key: api_key
Authorization: auth_token
```

We would use `DELETE` HTTP method as we are terminating a session.

Response:

```
200 OK
```

The response would be HTTP response code `200` indicating successful sign-out.

#### Subscribe

Request:

```
POST /api/v1/subscription
X-API-Key: api_key
Authorization: auth_token
```

We would be using `POST` HTTP method for subscribing as we are creating a new subscription. We would be passing the `auth_token` in `Authorization` header for authenticating the user.

Response:

```
201 Created
{
  subscription_id:
  plan_name:
  valid_till:
}
```

If user is able to subscribe successfully, we would return `201` HTTP response code along with `subscription_id`, `plan_name` and `valid_till` to render these in the UI.

Other possible failure HTTP response codes are listed below:

```
401 Unauthorized
400 Bad request
```

#### Unsubscribe

Request:

```
DELETE /api/v1/subscription
X-API-Key: api_key
Authorization: auth_token
```

We would be using `DELETE` HTTP method as we are cancelling the subscription.

Response:

```
200 OK
```

Once subscription is terminated successfully, we would return HTTP code `200`.

#### Get Videos

Request:

```
GET /api/v1/videos?page_id=<page_id>
X-API-Key: api_key
Authorization: auth_token
```

This API would be used to render the home page of the application once user logs in. This API would contain recommended videos for that user which would be determined using machine learning models. `page_id` would be used for [pagination](https://en.wikipedia.org/wiki/Pagination){:target="\_blank"} in API and `next_page_id` would be used for requesting results from next page.

Response:

```
200 OK
{
  page_id:
  next_page_id:
  videos: [
    {
      id:
      title:
      summary:
      url:
      watched_till:
    },...
  ]
}
```

The API would return HTTP status code `200` indicating a successful operation. The response would contain an array of video object. Some failure status codes are listed below:

```
401 Unauthorized
500 Bad request
429 Too many requests
```

HTTP status code `429` indicates that client has hit the rate limit and it should wait some time before making request again. This is mainly to prevent [Denial of Service](https://en.wikipedia.org/wiki/Denial-of-service_attack){:target="\_blank"} (DOS) attacks.

#### Search Videos

Request:

```
GET /api/v1/search?q=<query>&page_id=<page_id>
X-API-Key: api_key
Authorization: auth_token
```

This API would be used to search a video by title.

Response:

```
200 OK
{
  page_id:
  next_page_id:
  videos: [
    {
      id:
      title:
      summary:
      url:
      watched_till:
    },...
  ]
}
```

API response would include an array of video objects matching the search query.

#### Get Video

Request:

```
GET /api/v1/videos/:video_id
X-API-Key: api_key
Authorization: auth_token
```

This API would be used to play a particular video.

Response:

```
200 OK
{
  id:
  title:
  summary:
  url:
  watched_till:
}
```

API response would include a video object matching the `video_id`. Other failure HTTP status codes are mentioned below:

```
401 Unauthorized
404 Video not found
429 Too many requests
500 Internal server error
```

#### Upload Video

Request:

```
POST /api/v1/videos
X-API-Key: api_key
Authorization: auth_token
{
  title:
  summary:
  censor_rating:
  video_contents:
}
```

This API would be used to upload a video from Netflix backend.

Response:

```
202 Accepted
{
  video_url:
}
```

Once video is uploaded, API would respond with HTTP status code `202`, which would mean that video has been enqueued for asynchronous processing and quality checks. The result of the processing can sent to users via email or other alerting mechanisms.

Some of the HTTP status codes for failure scenarios are mentioned below:

```
401 Unauthorized
400 Bad request
500 Internal server error
```

#### Update watched till timestamp

Request:

```
PUT /api/v1/videos/:video_id/watched_till
X-API-Key: api_key
Authorization: auth_token
{
  watched_till:
}
```

This API would be used to update the timestamp till the user has watched a particular video. This would facilitate us to restart the video from same time stamp in future if user drops in between. We would be using `PUT` HTTP method as we are updating a resource on server.

Response:

```
200 OK
```

On successful update, API would return HTTP status code `200`. Some of the HTTP status codes for failure scenarios are mentioned below:

```
401 Unauthorized
400 Bad request
500 Internal server error
```

### Performance

We have incorporated couple of elements in our design which help immensely in improving the performance of our application. Let's look in to both of them.

#### Caching

We would be using an in-memory caching to reduce the number of hits to database. In-memory caches like Redis and Memcached cache the data from database in key-value pair.

From our application servers, before hitting the database we would be checking if data exists in cache. If it exists then we would return the value from there bypassing a database trip. However if data is not present in cache, we would hit the database, get the data and populate the same in cache too. Hence subsequent requests won't hit the database and get the data from cache itself. This caching strategy is known as write-around caching.

We would use Least Recently Used (LRU) eviction policy for caching data as it allows us to discard the keys which are least recently fetched.

#### CDN

[Content Delivery Network](https://en.wikipedia.org/wiki/Content_delivery_network){:target="\_blank"} is a geographically distributed network of servers which deliver content to users from geographically closest server. This reduces the latency and number of network hops thereby improving the performance of content delivery.

The use of CDN for delivering video would boost the performance as videos would be served from location near to users reducing the number of network hops. Also CDN can cache the video in memory for even faster delivery.

In-fact Netflix has build their own CDN network known as [Open Connect](https://openconnect.netflix.com/en/){:target="\_blank"} for delivering the videos. Open Connect uses specialized Open Connect Appliances (OCAs) embedded in ISP networks which are highly optimized for video delivery.

### Scalability

Our architecture is highly scalable owing to following facts:

#### Horizontal Scaling

We can add more application servers behind the load balancer to increase the capacity of the service. This is known as Horizontal Scaling and each service can be independently scaled horizontally in our design.

#### Database Replication

We would be using the relational database in [Master-Slave](<https://en.wikipedia.org/wiki/Replication_(computing)#DATABASE>){:target="\_blank"} configuration where writes will happen to Master and Reads from Slave. This will improve the performance of read queries as they won't be halted due to write locks on row. Though there would be a slight replication lag (~ few milliseconds) as data would be written to Master first and then propagated to Slave later on, it would be fine for this use case.

#### Database Sharding

Since our system is read intensive, we need to distribute our data to multiple servers to perform read/write operations efficiently. This can be achieved via Database Sharding.

We would be sharding the video metadata database using `video_id`. Our hash function will map each `video_id` to a random server where we can store the video metadata. To query for a particular `video_id`, service can determine the database server using same hash function and query for data. This approach will distribute our database load to multiple servers making it scalable.

#### Cache Sharding

Similar to Database Sharding, we can distribute our cache to multiple servers. In-fact Redis has out of box support for [partitioning](https://redis.io/topics/partitioning) the data across multiple Redis instances. Usage of [Consistent Hashing](https://en.wikipedia.org/wiki/Consistent_hashing){:target="\_blank"} for distributing data across instances will ensure that load is equally distributed if one instance goes away.

#### Search Database Sharding

Search Database, powering our full text search, can also be sharded across multiple servers. Elasticsearch comes with native support for sharding and replication. Sharding helps in improving the query runtime by running them in parallel against multiple shards.

### Security

Our system would be highly secure due to following:

#### HTTPS

We would be encrypting the traffic between client and server over HTTPS. This will ensure that no one in the middle is able to see the data, especially passwords.

#### Authentication

For each API request post log-in, would be doing authentication by checking the validity of `auth_token` in `Authorization` HTTP header. This will make sure that requests which originate from clients are legitimate.

### Resiliency

Our system would be highly resilient owing to following:

#### Replication

We would be replicating our database servers in Master-Slave configuration. If one of these nodes does down, other would take the place and service would continue functioning as expected. Even our cache and search database would have replicated shards which will add resiliency to the overall system.

#### Queuing

We would be using the queuing in our system for processing the uploaded videos. Hence if a worker dies, message in a queue won't be acknowledged and other worker would pick up the task again.

#### Load Balancing

Since we would be putting multiple servers behind a load balancer, there would be redundancy. Load Balancer will continuously be doing health check on servers behind it. If any server dies, load balancer will stop forwarding the traffic to it and remove it from cluster. This will make sure that requests don't fail due to a unresponsive server.

#### Geo-redundancy

We would be deploying exact replica of our services in data-centers across multiple geographical locations. This would ensure that if one data-center goes down due to some reason, the traffic could still be served from remaining data-centers.
