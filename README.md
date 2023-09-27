# Homelab
This is a catch-all for all of my homelab related things. I realized that as I figure things out, I want to try and document exactly how I implemented them so that future me wouldn't be screwed over. I will also add links to reference materials and source directories as needed.

---

_Update 09/27/23: I have continued working on my homelab, but for obvious reasons (general privacy stuff) I have not uploaded my docker compose here. I am working on converting all docker compose to ansible playbooks, for better secret management. Look for that coming in the next month or so._

---

## Resources
- [How to set up TrueNAS with Tailscale and Nextcloud](https://forum.level1techs.com/t/truenas-scale-ultimate-home-setup-incl-tailscale/186444)
- [Traefik and SSL Certificates](https://www.youtube.com/watch?v=liV3c9m_OX8)
    - [Repo with config from video](https://github.com/techno-tim/techno-tim.github.io/tree/master/reference_files/traefik-portainer-ssl)
- [Using Pi-hole w/ Tailscale](https://leonsteenkamp.co.za/posts/traefikproxyintro/)

# Basic Setup
I'm currently running everything on TrueNAS Scale. It works for me, since my homelab functions primarily as a NAS/backup server. It's running on some pretty aged hardware, but it does the job for now. 

For storage, I decided to get two 4 Tb HDDs. If I could go back, since my storage requirements are so low I would have gotten a 4 Tb SSD instead. It might have been a little more expensive, but for my usecase it would have been a better solution. 

## VMs
TrueNAS honestly kind of sucks in terms of virtualization. I wish I had gone with something like Proxmox instead, and virtualized TrueNAS. I followed [this Level1Techs tutorial](https://forum.level1techs.com/t/truenas-scale-ultimate-home-setup-incl-tailscale/186444) on how to get VMs to work properly on TrueNAS Scale.

I am currently running two small VMs - one that runs all of my Docker containers for services and another that is responsible for only Pi-hole. I originally had everything running in the same VM, but I moved to this setup to allow me to break my docker VM without crashing my DNS server (it made setting up Traefik a lot easier).

# Network and DNS
So funny story. I had some preliminary stuff running (TrueNAS, pi-hole, bitwarden), and then I had the thought 'hey, I want to have domain names for everything.' That was a dangerous thought, something that led to easily 10+ hours of work. But it was a lot of fun, and I learned a lot in the process.

## Attempt #1: Try to just use Pi-hole
Pi-hole is a DNS. So my first attempt at this set up was just trying to setup DNS entries for each of my docker containers. However, I learned that you in fact can't set DNS entries for ports, which means that if you have multiple docker containers on the same machine, you can't have DNS entry for each of them. Fun. So, I needed a way to point at my VM IP address, and have that domain name be directed to the port and resulting service. Enter the reverse proxy.

## Attempt #2: Use a reverse proxy (Traefik) w/o a domain name
For my noob level of understanding, a reverse proxy is a service that sits in front of your actual server(s), and directs requests to the proper location. For my goal (trying to get domain names for each of my services), this is exactly what I needed. 

After some google searching, I found [Traefik Proxy](https://traefik.io/traefik/). Now, for people who are more informed than me in the homelab/selfhosting community, you are probably asking why I chose this particular proxy. Because it is genuinely a pain in the ass to setup. Nginx probably would have been a better option for me, someone only a week into setting up selfhosted services and having no experience with networking. But alas, I had no idea it was a thing.

I primarily followed these tutorials form some fantastic Youtube homelabbers: [this video about SSL and Traefik by Techno Tim](https://www.youtube.com/watch?v=liV3c9m_OX8) and [this basic tutorial by Christian Lempa](https://www.youtube.com/watch?v=wLrmmh1eI94). I would have been totally lost without them. 

You will notice that both of these tutorials also set up SSL for your domain names, which is kind of nifty. Not so nifty if you have no idea about networking of SSL. My first attempt at using traefik was to ignore all the SSL stuff, and use just the DNS entries in Pi-hole (ie service.local). No dice. I had a really frustrating experience with it, because I couldn't even get the traefik dashboard up. And when I eventually did get the dashboard up (you have to set `dashboard:true` in the [traefik.yml](/docker/traefik/data/traefik.yml) and you have to set the flag `- "traefik.http.routers.traefik-secure.service=api@internal"` in the [docker-compose.yml](/docker/traefik/docker-compose.yml)), I still couldn't get DNS entries to resolve. I still am not 1000% sure why that was the case, outside of the fact that traefik possibly needs full domain names, not just DNS entries. After a lot of monkeying around, I gave in a purchased a domain name.

## Attempt #3: Traefik w/ a domain name
I purchased a Google domain name, and then followed the Techno Tim tutorial listed above. As a note, I set up Portainer first, and then set up Traefik in Portainer, and adjusted the Portainer docker-compose.yml until it worked with Traefik. 

With that tutorial, I was able to get domain names for all of my services and everything worked. I initally was trying to use the Let's Encrypt URL in the Christian Lempa tutorial, in combination with the stuff from Techno Tim. It was a mess, and above my current abilities. I ended up registering my domain with Cloudflare, which I didn't realize was free (hence my previous apprehension). Once I followed the Techno Tim tutorial word for word (no more monkeying around), I finally got everything to work. Well, almost everything. I had a couple of issues still. 

1. My SSL certificates were not being resolved by Cloudflare. This ended up being an issue with my DNS entries in Cloudflare. You have to have your entry (something like local.example.com) point at your _public_ IP address, and you have to forward port 443 on your router. Set your router to only accept requests from Cloudflare server IPs to maintain security. I was afraid that this would put myservices on the public web, but because I'm using Pi-hole as local DNS, none of my local services resolve outside of my local network. If I put the full sub-sub-domain in cloudflare, they would. So pro tip, don't do that.
2. I could not set password authentication on my Traefik dashboard. In the Traefik documentation, they specify that [passwords must be hashed](https://doc.traefik.io/traefik/middlewares/http/basicauth/). Did not realize that. I used [htpasswd](https://httpd.apache.org/docs/2.4/programs/htpasswd.html), and passed the file containing the username and hashed password to my [docker-compose.yml](/docker/traefik/docker-compose.yml). I wasn't able to just put the hashed password into the [docker-compose.yml](/docker/traefik/docker-compose.yml) - I think there was a problem with the quotes, but I'm not sure.
3. I use Tailscale as my VPN. Even when I forwarded my subnet with Tailscale, I could not get my domain names to resolve. I kept getting an SSL error. What I ended up doing was setting up an additional subdomain (ts.example.com) that in Pi-hole would point at the Tailscale IP address for Traefik. This was actually a blessing in disguise, because it meant that I could expose only particular services over VPN, adding an extra level of security to my setup. 

And with that, my initial networking was complete! It was a pain in the ass, but I learned a ton and I now have a really robust foundation that has made adding new services a piece of cake. 

# Other services

## Vaultwarden
I've been using Bitwarden for some time (at least 2 years), and I really love it. A couple of weeks ago I was watching a LTT video, and it was brought up that using iOS password manager on your phone is incredibly insecure since, if your phone is stolen and someone guesses your password, they will have access to _all_ of your passwords on your phone. No good. So, I migrated everything into Bitwarden, and it's been a lovely experience.

I migrated everyting now into Vaultwarden, becasue it has a smaller computational footpring than the official Bitwarden server for self-hosting. With my VPN, it makes it easy to keep everything up to date on my phone and laptop when I'm out and about. The other nice thing about Bitwarden is that passwords are stored locally, so it doesn't actually matter if you have constant access to the main server. I prefer to have it synced all the time though.

## Homer
I've been seeing dashboards of Youtubers here and there, and I kind of fell in love with them. The one I like the most was [Homer](https://github.com/bastienwirtz/homer), because of how customizable it is. I also just fell in love with [this theme by Walkx](https://github.com/walkxcode/homer-theme), which is what I'm currently using. If I have some time, I'm going to mess with some CSS and make my own. But for now, this is a beautiful alternative.

I set up two seperate dashboards - one for my home PC, and one for my laptop. My home one includes links to my homelab services, along with other links I normally had in my browser taskbar. It makes getting to things I need a lot more convenient, and I love it. My laptop dashboard is a truncated version of the home dashboard, and does not include any of my primary homelab management stuff (like portainer or traefik). I've included the icons I used for my services [here](/docker/homer/tools/). 

## Vikunja
For a couple of years now, I've been using a Notion dashboard that I found on Google for all of my TODOs. And it's been fine. It had a lot of stuff on it that I really didn't use very much, and it wasn't very convenient to use. So I was on the hunt for a dedicated TODO app, not just something that was cobbled together. Enter [Vikunja](https://vikunja.io/). On the selfhosting subreddit, it seems to have gotten a mixed response, but it's almost perfect for me. What I really wanted was the ability to organize tasks into buckets of Today, tomorrow, and next 7 days, along with different categories like work and personal. You can do this in Vikunja - I created an 'Inbox' list, labels for each of my categories and then filters for that fished out my tasks that were due Today, Tomorrow, and Next 7 Days. 

There is no iOS app for Vikunja, but if you pin the bookmark to your homepage it functions essentially the same. I've had no issues with it so far. 

# Conclusion
This is my homelab setup so far. I'm planning on upgrading my hardward in the near future so I can host a couple more VMs for development/testing. I also want to implement Prometheus/Loki/Graphana for monitoring my containers, along with [code-server](https://github.com/coder/code-server) for remote work. I'm also going to dabble with some Ansible to automate docker updates and possible container deployment when I get closer to migrating my current setup. I think I'm hosting what I want to right now, but I'm still exploring and learning!

