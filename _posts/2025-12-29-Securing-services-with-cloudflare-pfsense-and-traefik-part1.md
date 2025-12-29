---
layout: post
title: 'Securely hosting services with CloudFlare, pfSense and Traefik (Part 1)'
tags:
  - guides
  - pfsense
category: Network
---
I’ve exposed home services to the internet for years, and I’ve finally landed on a setup that balances security with practicality, it’s not necessarily the best if you are looking to follow the latest and greatest principles of Zero Trust .etc., but this guide does improve understanding of pfSense NAT, Cloudflare, DynamicDNS, Reverse Proxies and Docker, without turning a HomeLab into a second job.

Part 1 – Setting up CloudFlare, pfSense DynamicDNS, pfBlockerNG, Firewall Rules and Port Forwarding.

Part 2 (coming soon) – Setting up Traefik Reverse Proxy and sample Docker containers.

This is what I’m running:

*   Standard home broadband with a **dynamic public IP**
    
*   **pfSense 25.11** as the edge router/firewall.
    
*   **Dell PowerEdge T110** as a Docker host (old, reliable, and refuses to die)
    
*   **Traefik** as the reverse proxy for internal routing to containers
    
*   **Docker containers** hosting my services
    

The goal is:

1.  Make services reachable via friendly hostnames
    
2.  Keep DNS updated automatically as my IP changes
    
3.  Ensure inbound traffic **must** come via Cloudflare (no direct IP hits)
    
4.  Route everything cleanly to the right Docker service without endless manual config
    
5.  Implement security best practices and minimise complexity where possible.
    

At this point I assume you have a domain name, and you are using CloudFlare name servers.
