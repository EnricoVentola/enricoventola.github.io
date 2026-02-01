---
layout: post
title: 'Securely hosting services with CloudFlare, pfSense and Traefik (Part 1)'
tags:
  - guides
  - pfsense
category: Network
---






I’ve exposed home services to the internet for years, and I’ve finally landed on a setup that balances security with practicality, it’s not necessarily the best if you are looking to follow the latest and greatest principles of Zero Trust .etc., but this guide does improve understanding of pfSense NAT, Cloudflare, DynamicDNS, Reverse Proxies and Docker, without turning a HomeLab into a second job.

### **Part 1 – Setting up CloudFlare, pfSense DynamicDNS, pfBlockerNG, Firewall Rules and Port Forwarding.**

_Part 2 (cominag soon) – Setting up Traefik Reverse Proxy and sample Docker containers._

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

**Step 1 – Dynamic DNS**

**Step 1.1 – Cloudflare API**

With a dynamic public IP, manually updating DNS records is a non-starter. Plenty of DynDNS tools can do it, but I wanted to avoid bolting on yet another script or container if pfSense could handle it natively.  pfSense supports Dynamic DNS out of the box, and it can call a Cloudflare API to update DNS records automatically.

Login to CloudFlare, goto [API Tokens | Cloudflare](https://dash.cloudflare.com/profile/api-tokens).

Under **API Token Templates,**  select **Edit zone DNS** then **Use template.**

We should operate the principle of _Least Privilege(_[Least Privilege Principle | OWASP Foundation](https://owasp.org/www-community/controls/Least_Privilege_Principle) _)_, so lets go to **Zone Resources** and select only the DNS domain that we need.

Click **Create Token**, copy the token and save it for later.

**Step 1.2 – pfSense Package.**

Login to PfSense, goto **Services** > **Dynamic DNS** > **Add**.

If you already have an item in the list, it’s better to duplicate because it will copy an existing CloudFlare token oto a new record.

**Service Type:** Cloudflare

**Interface to monitor:** WAN (typically the you want to track)

**Hostname:**

            First box: Enter the hostname like “corp”.

            Second box: Enter the root domain like “contoso.com”.

**Cloudflare Proxy:** Enable (I’ll explain later)

**Username:** Use your CF account username

**Password:** Use the API token that we just created.

Click **Save & Force Update.**

You should see something like this:

![](data:image/png;base64,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)

The geen check means my IP address has synced to to a Cloudflare DNS A Record.

**Step 2 – CloudFlare Security**

With the record CloudFlare (CF) Proxy enabled, we get the benefits of DDoS protection, optimised caching and a host of other essential features. This also promotes the principle of _Edge Security_**.** CF is doing the heavy lifting outside of our traditional network boundary, so the traffic we eventually receive at the router should be safer. We can shift security policies to the edge, far away from our internal network. A common pitfall at this point; adding extra products that do the same thing, a bit like overlapping anti-virus products.

We’ll leave the CloudFlare config as standard for now, bout ideally enabling features like DNSSEC is good practice.

**Step 2.5 – Pfsense Security.**

If someone learns my Public IP, they can access my services by bypassing DNS and my Cloudlflare proxy, ideally I want only traffic that has come via the CF Proxy allowed through my router.

_PFBlockerNG_ (PFB) is a package that lets us, amongst many things, assign URLs and IPs to alias lists that we can then use in Firewall rules. PFB will continuosly update them.

Goto **System  > Package Manager > pfBlocker-NG-Devel > Install.**

Once installed, goto **Firewall > pfBlockerNG > IP > IPv4 > Add**

CloudFlare published their edge IP addresses (IPv6 too) at [https://www.cloudflare.com/ips-v4](https://www.cloudflare.com/ips-v4) or ips-v6

Configure it like this:

Also, the following:

**Action:** Alias Native – 'Alias' rules create a list and do nothing else. This enables a pfBlockerNG list to be used by name somewhere else on pfSense, like a firewall rule.

**Update frequency**: Once a day – A check on Wayback Machine reveals CF rarely update their IPv4 range, I got to 18 months before moving on. Ideally we could watch the CF RSS Feed for mentions of changes and call an API, but let’s park that for now. If their IPs do change, it could cause service disruptions and we’re only calling one small web page per day so it will suffice; rather not overcomplicate.

Click **Save IPv4 Settings,** then go back to the PFB main menu, select **Update > Run.** This will create and populate the list.

In Firewall > WAN: Create a new rule.

Select TCP/UDP, set sources type to “Address or Alias”, then start typing the name of the list that you created in the previous step, it will populate. Then set your port to 443 for HTTPS, the rest is optional.

You should end up with this.

**Step 3 – pfsense routing**

Now we need a way to forward user requests to the right place after they arrive to the router, but we can only use the hostname field as destination IP is always my public IP and we are using NAT. We need a reverse proxy.

HAProxy is built in to pfsense, but I find it clunky to use and crucially requires manual config for each of my docker containers, enter Traefik.

Since the router is not handling reverse proxy, we need to use DNAT to send all packets to the Traefik reverse proxy docker container:

In Firewall > NAT: Create a new rule.

Interface: WAN

Address Family: IPv4

Protocol: TCP/UDP

Source: Address or Alias

Type: Select the name of the list you created for me its “pfB\_ALLOWED\_PUBLIC\_INBOUND\_v4”

Destination: WAN Address

Destination port range: HTTPS for both

Redirect target IP: Alias or Address

Type: Enter the IP address of Traefik reverse proxy, for me it’s 10.1.40.36

Redirect target port: HTTP
