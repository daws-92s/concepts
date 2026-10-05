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

**The actual lookup path a browser takes**, in order — each step only runs if the one before it didn't already know the answer:
```
Browser cache → OS → OS cache → DNS resolver → Root servers → TLD registry → Nameservers → A record
```
Most lookups never reach the root servers at all — they're answered from a cache (the browser's, the OS's, or the resolver's) long before that, which is the whole point of caching: skip the full trip when a recent answer is already on hand.

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

## DNS Record Types
A domain can hold several kinds of DNS records, each pointing to a different kind of destination:

| Record | Points to | Example use |
|---|---|---|
| **A** | An IPv4 address | `daws92s.online → <frontend-public-ip>` — the most common record, points a domain straight at a server |
| **AAAA** | An IPv6 address | Same idea as A, for IPv6 |
| **CNAME** | Another domain name (an alias) | `www.daws92s.online → daws92s.online` |
| **NS** | The nameservers responsible for the domain | Points to whichever provider is actually managing the records — the registrar's own nameservers, or a different one like AWS Route 53, Cloudflare, etc. |
| **MX** | A mail server | Routes email sent to the domain to the right mail provider |
| **TXT** | Arbitrary text | Domain ownership verification, SPF/DKIM records for email |
| **SOA** | Start of Authority — metadata about the domain's zone | Which nameserver is the primary source of truth, plus settings like TTL defaults |

## TTL (Time to Live)
Every DNS record has a **TTL** — how long a resolver is allowed to treat a cached answer as still valid before it has to ask again. It's a direct trade-off:

| | High TTL (e.g. 1 day) | Low TTL (e.g. 1 minute) |
|---|---|---|
| Lookup speed | Faster — most requests hit a cache, few reach the real nameservers | Slower — more requests have to re-check with the nameservers |
| Change propagation | Slow — an IP change can take up to the full TTL to reach everyone | Fast — a change is visible almost everywhere within the TTL |

This matters most when you're about to change a record — e.g. moving a domain from an on-premise server's IP to a new cloud IP. The common pattern: lower the TTL (e.g. to 1 minute) a day or two *before* the change, let that propagate, make the IP change (which now spreads fast since the TTL is already low), confirm everything's working, then raise the TTL back to its normal value.

Skipping that step causes exactly the kind of problem TTL is meant to prevent: a visitor whose resolver cached the old record under yesterday's long TTL keeps getting routed to the old IP until that TTL finally expires — even though the record was "already changed."

## Pointing a Domain at AWS (Route 53)
Buying a domain from a registrar doesn't mean that registrar has to manage its DNS records — they can be delegated elsewhere. A common setup for a domain whose server lives on AWS:
1. Create a **hosted zone** for the domain in AWS Route 53 — AWS hands back a set of its own nameservers for that domain.
2. Go back to the registrar and update the domain's **NS record** to point at those AWS nameservers instead of the registrar's default ones.
3. Once that change propagates, Route 53 becomes authoritative for the domain — any record created there is what the world actually sees when it looks up the domain.
4. Add an **A record** in Route 53 pointing the domain at the server's public IP.

From then on, `http://daws92s.online` resolves straight to the server — nobody needs to remember or share the IP, and if the server's IP ever changes, only that one A record needs updating rather than every place the IP was shared.

## Common problems and how to solve them
A common misconception is that the registrar "owns" your DNS — it doesn't. The registrar just manages which nameservers the registry has on file for your domain. You can register a domain at one registrar and point its nameservers at a completely different provider (Cloudflare, AWS Route 53, etc.) to actually manage the DNS records.

Another common confusion: thinking a domain name *is* the server. It isn't — it's just a pointer. Changing a DNS record (e.g. an A record pointing to a new IP) doesn't move or change your server; it just changes where the name resolves to, and that change can take time to propagate depending on DNS caching (TTL).

## Key takeaways
- DNS exists because computers route by IP, not by name — it's purely a name-to-IP translation layer.
- The hierarchy is root servers → TLD registry → registrar → your nameservers → the IP address.
- Registry vs registrar: the registry (e.g. Verisign for `.com`) is the authoritative record-keeper for a TLD; the registrar (e.g. GoDaddy) is who you actually buy from — a retailer, not the record-keeper.
- ICANN oversees the whole system but is an independent nonprofit, not a government body, despite having originated under U.S. government oversight decades ago.
- Buying a domain doesn't give you a server — it gives you a name you can point at one, and that pointer is exactly what DNS records control.
- Different record types do different jobs: **A** points a domain at an IP, **CNAME** aliases one domain to another, **NS** says who manages the records, **MX** routes email.
- A domain's DNS doesn't have to stay with the registrar it was bought from — updating its NS record lets another provider (e.g. AWS Route 53) take over managing its records entirely.
- TTL controls the trade-off between lookup speed and how fast a change propagates — lower it before a planned IP change, then raise it back once the change has settled.

See also: [07-3-tier.md](07-3-tier.md)
