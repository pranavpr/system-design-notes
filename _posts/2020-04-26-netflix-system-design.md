---
layout: post
title: Netflix System Design
date: 2020-04-26 21:05 +0530
categories:
  - System Design
  - Netflix
---

Let's design a on-demand video streaming service like Netflix where users should be able to stream and watch videos which are available on the platform.

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

### Detailed Component Design

{% include image.html image="netflix-system-design.png" alt="Netflix System Design" %}

### Database Schema

{% include image.html image="netflix-database-schema.png" alt="Netflix Database Schema" %}
