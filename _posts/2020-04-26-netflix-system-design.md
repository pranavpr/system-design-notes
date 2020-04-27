---
layout: post
title: Netflix System Design
date: 2020-04-26 21:05 +0530
categories:
  - System Design
  - Netflix
cover: netflix-system-design-cover.jpg
---

Netflix is one the largest on-demand video streaming service over the internet. Netflix allows users to stream and watch videos which are available on its platform. Let try to build a system similar to Netflix.

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

{% include image.html image="netflix-system-design.png" alt="Netflix System Design" %}

### Database Schema

{% include image.html image="netflix-database-schema.png" alt="Netflix Database Schema" %}

### APIs

Different services will expose REST APIs to interact with clients or other services. Below are some major APIs:

#### User Sign-up

Request:

```
POST /api/v1/users
X-API-Key:
{
  name:
  email:
  password:
}
```

Response:

```
201 Created
{
  message:
}
```

#### User Sign-in

Request:

```
POST /api/v1/signin
X-API-Key:
{
  email:
  password:
}
```

Response:

```
200 OK
{
  auth_token:
}
```

#### Subscribe

Request:

```
POST /api/v1/subscription
X-API-Key
Authorization: auth_token
```

Response:

```
201 Created
{
  subscription_id:
  plan_name:
  valid_till:
}
```

401 Unauthorized  
400 Bad request

#### Unsubscribe

Request:

```
DELETE /api/v1/subscription
X-API-Key:
Authorization: auth_token
```

Response:

```
200 OK
```

#### Get Videos

Request:

```
GET /api/v1/videos
X-API-Key:
Authorization: auth_token
```

Response:

```
200 OK
{
  [
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

401 Unauthorized  
500 Bad request  
429 Too many requests

#### Search Videos

Request:

```
GET /api/v1/search?q=<query>
X-API-Key:
Authorization: auth_token
```

Response:

```
{
  [
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

#### Get Video

Request:

```
GET /api/v1/videos/:video_id
X-API-Key:
Authorization: auth_token
```

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

401 Unauthorized  
500 Bad request  
429 Too many requests

#### Upload Video

Request:

```
POST /api/v1/videos
X-API-Key:
Authorization: auth_token
{
  title:
  summary:
  censor_rating:
  video_contents:
}
```

Response:

```
202 Accepted
{
  video_url:
}
```

401 Unauthorized  
400 Bad request  
500 Internal server error
