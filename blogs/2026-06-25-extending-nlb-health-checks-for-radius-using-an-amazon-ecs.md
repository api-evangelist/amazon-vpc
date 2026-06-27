---
title: "Extending NLB health checks for RADIUS using an Amazon ECS witness"
url: "https://aws.amazon.com/blogs/networking-and-content-delivery/extending-nlb-health-checks-for-radius-using-an-amazon-ecs-witness/"
date: "2026-06-25"
author: "Tim Franklin"
feed_url: "https://aws.amazon.com/blogs/networking-and-content-delivery/feed/"
---
Network Load Balancer health checks confirm that a RADIUS server is reachable, not that it can authenticate a user, so a server with a failed identity store keeps receiving traffic. This post walks through an open-source reference solution that closes the gap with a single Amazon ECS witness that runs application-layer RADIUS probes and reconciles NLB target group membership directly.
