# Why Cloud Migration Is Happening Now

## What this is / why it matters
Running your own servers (on-premise) means you own every part of the problem: buying the hardware, housing it, powering it, cooling it, connecting it to a network, and keeping it physically secure. Cloud providers (AWS, Azure, GCP) already did all of that at massive scale, and they let you rent a slice of it. That's the whole pitch: skip the upfront cost and lead time, pay only for what you use, and scale up or down in minutes instead of months.

## How it works

**On-premise, the hard way:**
- Buy physical servers
- Rent or build a space to house them (like a warehouse)
- Set up power, cooling (AC), and network connections
- Install the OS yourself
- Handle physical security and surveillance
- Result: high upfront cost, long setup time, and no guarantee of high availability unless you build all the redundancy yourself

**Cloud, the easier way:**
- Spin up a server in minutes
- Pay only for what you use (no huge upfront purchase)
- The provider already handles the building, power, cooling, and physical security

**Regions and Availability Zones (AZs):**
- A **region** is a geographic area a cloud provider operates in (example: Mumbai, N. Virginia).
- An **Availability Zone** is one or more physically separate data centers within a region, each with its own power and cooling, so a failure in one doesn't take down the others.
- AWS guarantees at least **three** AZs per region (not fewer) so your application can be spread across independent failure points and stay up even if one AZ has a problem.
- Regions close to your users respond faster (lower latency). A user in Hyderabad hitting a Hyderabad-region server gets a quicker response than hitting a server in the US, simply because the data has less distance to travel.

**Setting up a cloud account (AWS, India context):** a few practical details that trip people up:
- Use a private debit/credit card with international transactions enabled
- The billing name should match your bank account name
- A PAN card is mandatory for account verification
- Use an email and phone number you haven't used for a previous AWS account (new accounts often get promotional free-tier credit; reused details may not qualify)

## Common problems and how to solve them
The core problem with on-premise is that cost and time are both upfront and both large — you pay for capacity you might not need yet, and you wait weeks for hardware to arrive before you can even start. Cloud fixes this by turning capital expense into operating expense: you provision only what you need right now, and change it later without buying anything new.

The other problem on-premise doesn't solve well is availability — if your one data center goes down, you're down. Spreading a cloud deployment across multiple AZs (and optionally multiple regions) means a single data center failure doesn't take your application offline.

## Key takeaways
- Cloud isn't cheaper because the hardware is magically free — it's cheaper because you stop paying for idle capacity and stop carrying the fixed cost of buildings, power, and physical security.
- A region is a location; an AZ is an independent data center within that location. Always design for multiple AZs if you care about uptime.
- Pick a region close to your users to reduce latency, not just because it's the default.
- "Pay as you use" is the real shift from on-premise — cost scales with actual usage instead of being locked in by a purchase decision made months earlier.
