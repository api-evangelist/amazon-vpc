---
title: "Best Practices for TCP Connection Management on EC2"
url: "https://aws.amazon.com/blogs/networking-and-content-delivery/best-practices-for-tcp-connection-management-on-ec2/"
date: "2026-05-22"
author: "Matt Lehwess"
feed_url: "https://aws.amazon.com/blogs/networking-and-content-delivery/feed/"
---
With sixth-generation Nitro (Nitro V6) instances, launched in June 2025, the default TCP connection tracking idle timeout changed from 432,000 seconds (5 days) to 350 seconds. Applications that hold idle connections open for long periods, such as database connection pools, Internet of Things (IoT) telemetry, and persistent microservice connections, may experience unexpected connection drops after […]
