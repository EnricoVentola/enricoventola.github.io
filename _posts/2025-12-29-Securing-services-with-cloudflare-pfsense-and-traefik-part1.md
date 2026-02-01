---
layout: post
title: 'Securely hosting services with CloudFlare, pfSense and Traefik (Part 1)'
tags:
  - guides
  - pfsense
category: Network
---
I’ve exposed home services to the internet for years, and I’ve finally landed on a setup that balances security with practicality, it’s not necessarily the best if you are looking to follow the latest and greatest principles of Zero Trust .etc., but this guide does improve understanding of pfSense NAT, Cloudflare, DynamicDNS, Reverse Proxies and Docker, without turning a HomeLab into a second job.

**Part 1 – Setting up CloudFlare, pfSense DynamicDNS, pfBlockerNG, Firewall Rules and Port Forwarding.**

_Part 2 (coming soon) – Setting up Traefik Reverse Proxy and sample Docker containers._

This is what I’m running:

•    Standard home broadband with a dynamic public IP
•    pfSense 25.11 as the edge router/firewall.
•    Dell PowerEdge T110 as a Docker host (old, reliable, and refuses to die)
•    Traefik as the reverse proxy for internal routing to containers
•    **Docker containers** hosting my services

The goal is:

1.    Make services reachable via friendly hostnames
2.    Keep DNS updated automatically as my IP changes
3.    Ensure inbound traffic must come via Cloudflare (no direct IP hits)
4.    Route everything cleanly to the right Docker service without endless manual config
5.    Implement security best practices and minimise complexity where possible.

At this point I assume you have a domain name, and you are using CloudFlare name servers.

**1 – Dynamic DNS**
**1.1 – Cloudflare API**

With a dynamic public IP, manually updating DNS records is a non-starter. Plenty of DynDNS tools can do it, but I wanted to avoid bolting on yet another script or container if pfSense could handle it natively.  pfSense supports Dynamic DNS out of the box, and it can call a Cloudflare API to update DNS records automatically.

Login to CloudFlare, goto [API Tokens | Cloudflare](https://dash.cloudflare.com/profile/api-tokens).

Under **API Token Templates,**  select **Edit zone DNS** then **Use template.**

We should operate the principle of _Least Privilege(_[Least Privilege Principle | OWASP Foundation](https://owasp.org/www-community/controls/Least_Privilege_Principle)_)_, so lets go to **Zone Resources** and select only the DNS domain that we need.

Click **Create Token**, copy the token and save it for later.

**1.2 – pfSense Package.**

Login to PfSense, goto **Services** > **Dynamic DNS** > **Add**.

If you already have an item in the list, it’s better to duplicate because it will copy an existing CloudFlare token into a new record.

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

![PFB.png]({{site.baseurl}}/_media/PFB.png)![PFB.png]({{site.baseurl}}/assets/img/PFB.png)

The green check means my IP address has synced to to a Cloudflare DNS A Record.

**2 – CloudFlare Security**

With the CloudFlare (CF) Proxy enabled, we get the benefits of DDoS protection, optimised caching and a host of other stuff. We’re promoting the principle of _Edge Security_**.** CF is doing the heavy lifting, shifting security policies to the edge, far away from our traditional network boundary, so the traffic we eventually receive at the router _should_ be safer.

We’ll leave the CloudFlare config as standard for now, but enabling features like DNSSEC is good practice.

**2.5 – Pfsense Security.**

If someone learns my Public IP, they can access my services by bypassing DNS and the Cloudlflare proxy, ideally I want only traffic that has come via the CF Proxy allowed through my router.

_PFBlockerNG_ (PFB) is a package that lets us, amongst many things, assign URLs and IPs to alias lists that we can then use in Firewall rules. PFB will update them as a cron job.

Goto **System  > Package Manager > pfBlocker-NG-Devel > Install.**

Once installed, goto **Firewall > pfBlockerNG > IP > IPv4 > Add**

CloudFlare published their edge IP addresses (IPv6 too) at [https://www.cloudflare.com/ips-v4](https://www.cloudflare.com/ips-v4) or ips-v6

Configure it like this:

Also, the following:

**Action:** Alias Native – 'Alias' rules create a list and do nothing else. This enables a pfBlockerNG list to be used by name somewhere else on pfSense, like a firewall rule.

**Update frequency**: Once a day – A check on Wayback Machine reveals CF rarely update their IPv4 range, I got to 18 months before moving on. Ideally, we could watch the CF RSS Feed for mentions of changes and call an API, but let’s park that for now. If their IPs do change, it could cause service disruptions and we’re only calling one small web page per day so it will suffice; rather not overcomplicate.

Click **Save IPv4 Settings,** then go back to the PFB main menu, select **Update > Run.** This will create and populate the list.

In Firewall > WAN: Create a new rule.

Select TCP/UDP, set sources type to “Address or Alias”, then start typing the name of the list that you created in the previous step, it will populate. Then set your port to 443 for HTTPS, the rest is optional.

You should end up with this.

**3 – pfSense routing**

Now we need a way to forward user requests to the right place after they arrive to the router. One aspect that differenties the service we need is the hostname, but we can’t use that on the firewall, so we need a reverse proxy.

HAProxy is built in to pfsense, but I find it clunky to use and crucially doesn’t integrate with Docker, that’s annoying because when I change or spin up new containers, It’ll pain me to fiddle with the config again, Traefik is the answer for me.

First, we need to forward all public traffic to the Traefik machine that we configure in Part 2.

DNAT is the way to go:

In Firewall > NAT: Create a new rule.
Interface: WAN
Address Family: IPv4
Protocol: TCP/UDP
Source: Address or Alias

Type: Select the name of the list you created above
“pfB\_ALLOWED\_PUBLIC\_INBOUND\_v4”

Destination: WAN Address
Destination port range: HTTPS for both
Redirect target IP: Alias or Address
Type: Enter the IP address of Traefik reverse proxy, for me it’s 10.1.40.36
Redirect target port: HTTPS
