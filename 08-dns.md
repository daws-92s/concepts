# DNS: How Domain Names Actually Work

## What this is / why it matters
Computers only route traffic using IP addresses — they have no idea what "google.com" means. Humans, on the other hand, can't remember `142.250.183.142` but can easily remember "google.com." DNS (Domain Name System) is the translation layer that bridges that gap, and understanding how it's structured explains what actually happens when you buy a domain and point it at your server.

## How it works

**The basic translation:**
```
facebook.com  →  104.104.56.87
```
A browser can't connect to a name — somewhere along the way, that name has to be resolved to an IP address before a connection can be made.

**The hierarchy — reading a domain name from right to left:**
```
joindevops   .   com
 (name)        (TLD)
```
- **TLD (Top-Level Domain)** — the last part: `.com`, `.in`, `.online`, `.org`, `.ai`, `.edu`, `.net`, `.uk`
- Each TLD has one **registry** — the organization that actually manages that TLD's database. `.com` and `.net` are run by Verisign; `.in` is run by NIXI (India's national internet registry); `.ai` is run by the government of Anguilla, since `.ai` is technically Anguilla's country-code TLD.
- A **registrar** (GoDaddy, Namecheap, Hostinger, Cloudflare, AWS, etc.) is who you actually buy a domain *from* — think of them as retailers/resellers, while the registry is more like the wholesaler/record-keeper for that TLD.

**Root servers and ICANN:**
Above every TLD sit the **root servers** — 13 well-known root server addresses (served from many physical locations worldwide, not 13 single machines) that know which organization manages which TLD. If a DNS lookup can't find an answer anywhere else, it eventually asks a root server "who manages `.com`?" and gets pointed to the right registry.

**ICANN** (Internet Corporation for Assigned Names and Numbers) is the nonprofit that oversees this entire system — the root zone, TLD policy, and the registrars allowed to sell domains. It isn't a government agency; it's an independent nonprofit, though it originated under oversight from the U.S. Department of Commerce and became fully independent of that oversight in 2016.

## What happens when you buy a domain
1. You go to a **registrar** (GoDaddy, Namecheap, etc.) and search for a domain, e.g. `joindevops.com`.
2. The registrar checks with the TLD's **registry** whether it's already taken.
3. If it's free, you provide your name, contact details, and payment to the registrar.
4. The registrar registers the domain and updates the TLD's registry with which **nameservers** manage that domain — usually the registrar's own nameservers, unless you point it elsewhere (e.g. to Cloudflare, or to your own DNS host).
5. From then on, anyone looking up `joindevops.com` gets routed: root servers → `.com` registry → your nameservers → the actual IP address you've configured.

The registrar earns a commission for handling the sale and paperwork on behalf of the registry — registries don't sell directly to the public.

## Common problems and how to solve them
A common misconception is that the registrar "owns" your DNS — it doesn't. The registrar just manages which nameservers the registry has on file for your domain. You can register a domain at one registrar and point its nameservers at a completely different provider (Cloudflare, AWS Route 53, etc.) to actually manage the DNS records.

Another common confusion: thinking a domain name *is* the server. It isn't — it's just a pointer. Changing a DNS record (e.g. an A record pointing to a new IP) doesn't move or change your server; it just changes where the name resolves to, and that change can take time to propagate depending on DNS caching (TTL).

## Key takeaways
- DNS exists because computers route by IP, not by name — it's purely a name-to-IP translation layer.
- The hierarchy is root servers → TLD registry → registrar → your nameservers → the IP address.
- Registry vs registrar: the registry (e.g. Verisign for `.com`) is the authoritative record-keeper for a TLD; the registrar (e.g. GoDaddy) is who you actually buy from — a retailer, not the record-keeper.
- ICANN oversees the whole system but is an independent nonprofit, not a government body, despite having originated under U.S. government oversight decades ago.
- Buying a domain doesn't give you a server — it gives you a name you can point at one, and that pointer is exactly what DNS records control.

See also: [07-3-tier.md](07-3-tier.md)
