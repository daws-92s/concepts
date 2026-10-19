# AWS CLI and Programmatic Access

## What this is / why it matters
Everything doable in the AWS Console (the web UI) can also be done from the command line or from code, using the AWS CLI or an AWS SDK (Software Development Kit). This matters for automation — a script or pipeline can create and manage infrastructure without anyone clicking through the console by hand.

## How it works

### Console vs CLI/SDK
Launching an EC2 instance from the console means filling in a form: a name, an AMI (the OS image), an instance type, a security group, storage (hard disk), and a key pair (for SSH access). The CLI equivalent is a single command, with each of those same choices passed as a flag:

```bash
aws ec2 run-instances \
    --image-id ami-0220d79f3f480ecf5 \
    --instance-type t3.micro \
    --security-group-ids sg-043444d172056acbd \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=roboshop-mongodb}]'
```

- `--image-id` — which AMI (OS image) to launch from.
- `--instance-type` — the instance size (CPU/RAM), e.g. `t3.micro`.
- `--security-group-ids` — which security group(s) (firewall rules) to attach.
- `--tag-specifications` — labels (like `Name`) attached to the resource, so it's easy to find later.

### Order matters: create the security group first
A security group has to already exist before an instance can be launched into it. The usual order is: create the security group first, then pass its ID (`sg-...`) when launching the instance.

### The CLI is a layer over the SDK
The AWS CLI is itself built on the AWS SDK — the same programmatic interface available to actual application code (Python, Java, Node.js, and so on) for calling AWS services directly, instead of typing commands by hand.

## Common problems and how to solve them
- **`run-instances` fails because the security group doesn't exist yet** — create the security group first, then reference its ID; it can't be launched into something that isn't there.
- **Mixing up `--image-id` and `--instance-type`** — the AMI decides *which operating system/image* boots up; the instance type decides *how much CPU/RAM* it gets. They're independent choices.

## Key takeaways
- Anything doable through the AWS Console has a CLI/SDK equivalent — this is what makes infrastructure automatable.
- `aws ec2 run-instances` takes the same decisions as the console's launch wizard (AMI, instance type, security group, tags), just as command-line flags.
- Create a dependent resource (like a security group) before the resource that references it.
- The AWS CLI is a thin layer over the AWS SDK — both ultimately talk to the same AWS APIs.

See also: [02-why-cloud-migration.md](02-why-cloud-migration.md), [13-shell-scripting.md](13-shell-scripting.md)
