---
weight: 4
title: "Terraform, Talos, and the Art of the Disposable Server"
date: 2026-05-24T17:40:49-06:00
lastmod: 2026-05-24T16:45:40+08:00
draft: true
description: "This article describes how to deploy Talos Linux using Terraform on AWS."
featuredImage: "/images/talos-banner.webp"
images: ["/images/talos-banner.webp"]

tags: ["Kubernetes", "Linux", Terraform]
categories: ["Cloud"]

lightgallery: true
---

<!--more-->

## In the land of AWS EKS, why deploy Talos Linux?

I’ve been running Talos Linux in my homelab for over two years now. Some people might say it is overkill and part of me understands that sentimnt and another part of me thinks those nay sayer's don't understand how to operate Kubernetes and that it actually provides alot of convience. That opinion deserves a whole other blog post. Talos makes managing and deploying Kubernetes a breeze by representing a fundamental shift in how we think about infrastructure. 

Instead of the traditional Kubernetes deployment on top of an operating system, Talos made Kubernetes the operating system. After all, if Kubernetes is the operating system of the cloud, why are we still managing Linux distros underneath it? Sidero Labs has essentially stripped away everything unnecessary, leaving only what’s required to run containers reliably. To be honest, I’m surprised it isn’t the defacto Kubernetes deployment everywhere. Maybe there are edge cases I haven’t hit yet. Having deployed Talos on proxmox in my homelab I thought it would be a good exercise in what configuraion is needed to get Talos up and running in AWS. Before I dig into the AWS infrastructure side of things I want to discuss what I like about Talos, primarlily the immutability/ephermal nature and security of the operating system.

### Immutability: From Pets to Cattle

In the DevOps space, we talk a lot about "Pets vs. Cattle" In the past, you treat your servers like pets: you name them, you nurse them back to health when they're sick, and you manage their individual updates and user configurations. But with Talos, we’re strictly in the "Cattle" business. These servers are ephemeral. You aren't a parent; you’re a butcher. Need a security update? You don’t patch the running server you send it to the slaughterhouse and roll out a new one. Need to add a new configuration? Spin up a new instance and tear down the old one. 

By automating this, you ensure your environment is consistent, repeatable, and most importantly completely disposable.This is where Talos really shines. It’s immutable by design. For example, I need the i915 extension to pass my onboard GPU through to Kubernetes for Plex transcoding. You can’t just "install" that once the system is live. Instead, Talos gives you the [Talos Factory](https://factory.talos.dev/) to bake those extensions directly into the image before you deploy. Need a change? You don’t log in and tweak settings you just update your image and redeploy. There is something very satisfying about destorying architecutre and having it spun back up in minutes. 

### Security by Omission: Small Footprint, Massive Protection

When it comes to security, Talos Linux takes a sledgehammer to the traditional attack surface. Or maybe it’s more accurate to say it takes a scalpel, because it ships with fewer than 50 binaries. The entire distribution is a tiny 100MB. Less bloat means faster runtimes and less Common Vulnerabilities and Exposures (CVE). We are talking about a boot time of around one minute and a fully functioning cluster up in less than five. WOW! 

Because immutability is baked directly into its DNA, you don't need a package manager. But the real game changer is how Sidero Labs handled remote access, they completely ditched SSH. Think about your typical Linux server. You have to expose port 22, manage SSH keys, and configure firewalls. With Talos, there is no SSH. There isn't even a shell to log into. Instead, you manage the entire OS remotely using `talosctl`, which operates via gRPC over mTLS. So we have a low latencey connection using utilizing a certificate generated right after the OS bootstraps. This is a huge win for security.

For a long time, certain highly regulated industries couldn't touch Talos because of strict compliance and auditing rules. However, Sidero Labs knocked down that barrier by introducing [FIPS-140-3 compliant](https://www.siderolabs.com/blog/talos-linux-fips-140-3-compliant-builds/) builds back in September 2025. This means Talos now officially meets the stringent data security standards set by the U.S. National Institute of Standards and Technology (NIST), opening the door for government, finance, and healthcare tech stacks to finally run a Talos cluster.

## The Philosophy: Keeping AWS Simple (and Cheap)

This isn’t going to be another tutorial on how to build Terraform modules from scratch. There are plenty of guides across the internet that setup already. Instead, I want to provide a highlevel overview of the goals I had in mind for each module. When it comes to building architecture in the cloud, my philosophy is simple: use as few cloud-specific services as possible. Give me compute in its purest form, just an EC2 instance. As a Linux administrator, I’m comfortable managing servers. Tools like Ansible and Bash/user-data, give me the freedom to spin up an instance exactly how I want it, leaving no doubt about what is happening under the hood. Compare that to certain native AWS services that feel like an expensive black box and require a massive time investment just to learn proprietary configurations you can't use anywhere else. Honestly, that’s a massive reason why I fell in love with Talos and Kubernetes in general. 

Kubernetes already has everything you need to manage workloads, handle rollouts, and monitor applications. Plus, I can deploy that exact same cluster across multiple clouds without changing my workflow. Take AWS Systems Manager (SSM) as an example. AWS heavily pushes you to use SSM for remote machine access, but the whole process feels clunky. It forces you to memorize long commands with multiple flags that require constant Googling. Sure, you could create a bash alias, but you shouldn't have to jump through hoops just to talk to your infrastructure. To simplify things, I took matters into my own hands. I deployed a tiny EC2 instance running Tailscale as a subnet router. It was incredibly simple, required almost zero setup, and instantly gave me secure access to every single machine inside the VPC. 

### Building the Foundation: VPCs, Subnets, and Routing

[Networking Module](https://github.com/jgrove90/talos-foundry/tree/main/modules/network) 

Every cloud project at its core is a networking project. At least that's been my experience. No matter what you're building, sooner or later you find yourself staring at route tables, CIDR blocks, and subnet boundaries wondering where your afternoon went. The Virtual Private Cloud (VPC) serves as the foundation for everything else in the environment. Before Kubernetes can schedule a pod, before Talos can bootstrap a cluster, and before Tailscale can provide secure access, the network needs to exist.

For this project, I wanted a reusable VPC module that could be deployed repeatedly without requiring manual configuration. The module automatically creates public and private subnets across multiple Availability Zones, configures route tables and associations, and manages Internet Gateway connectivity. One area where I intentionally deviated from the standard AWS architecture was NAT. AWS NAT Gateways are fantastic from an operational perspective because they're managed and highly available. They're also surprisingly good at generating monthly charges. Since this project is intended for learning, experimentation, and personal use, I opted to use the `fck-nat` Terraform module instead. It provides NAT functionality at a fraction of the cost while remaining perfectly adequate for this use case, and honestly it wouldn't take much more effort to make it highly availble for production workloads.

### Security Without Making Operations Miserable

[Security Module](https://github.com/jgrove90/talos-foundry/tree/main/modules/security) 

Security is one of those topics where people tend to fall into two camps. The first group opens everything to `0.0.0.0/0` and promises they'll lock it down later. The second group builds a security model so restrictive that even the administrators can't figure out how to access their own systems. I wanted something in the middle: secure by default, but still practical to operate.

The foundation starts with IAM. Oh the wonderful world of IAM. Rather than embedding credentials on instances or passing around access keys, every node receives an IAM role with only the permissions it actually needs. AWS handles credential rotation behind the scenes, which means there are no long-lived secrets sitting on servers waiting to become tomorrow's security incident.

From there, the cluster is segmented into dedicated security groups for control plane nodes, worker nodes, and management infrastructure. Communication between components is intentionally restricted to the ports required for Kubernetes and Talos to function. The control plane can talk to the workers, workers can talk to the API server, and nodes can communicate with one another when necessary but that's about it. If traffic isn't required for the cluster to operate, it probably isn't allowed.

I also wanted flexibility when it came to outbound traffic. Some environments are perfectly fine with unrestricted egress, especially in development. Others require much tighter controls. Rather than forcing a single approach, the module supports both models. You can allow standard outbound access or enable stricter egress policies that limit what nodes can reach outside the VPC.

Remote administration follows the same philosophy. Instead of exposing management services directly to the internet, I rely on AWS Systems Manager and Tailscale. SSM provides a secure way to access instances during break the glass scenerios, while Tailscale gives me private connectivity to the cluster without opening public management ports. The end result is that the Kubernetes API, Talos API, and internal services remain private while still being accessible from authorized devices.

### Control Plane First, Workers Later

[Compute Module](https://github.com/jgrove90/talos-foundry/tree/main/modules/compute) 

Once the networking and security foundations were in place, it was finally time to deploy some actual compute. This ended up being one of those areas that seems simple on the surface but quickly turns into a balancing act between flexibility, cost, and operational simplicity. For this deployment, I chose to focus exclusively on the control plane nodes. Worker nodes are intentionally absent. Before anyone starts sharpening their pitchforks, that's by design. I plan to use [Karpenter](https://karpenter.sh/) to handle worker node provisioning and autoscaling, which deserves its own dedicated blog post. 

The control plane nodes are the brains of the operation. They host etcd, maintain cluster state, and keep Kubernetes functioning. If the control plane disappears, you're going to have a bad day. Because of that, I wanted a deployment pattern that could survive an Availability Zone failure while remaining flexible enough to support different environments and workloads.

Rather than hardcoding infrastructure decisions into the Terraform resources, everything is driven through variables. Need static IP addresses? Supported. Want AWS to assign them dynamically? That's fine too. Running On-Demand instances in production? Easy. Looking to save money with Spot Instances in a development environment? Also supported.

### Bootstrapping Talos Without the Click-Ops

[Talos Bootstrap Module](https://github.com/jgrove90/talos-foundry/tree/main/modules/talos-bootstrap) 

Getting virtual machines deployed is actually the easy part. A Kubernetes cluster isn't a Kubernetes cluster just because a few EC2 instances are running. Somebody still has to generate cluster secrets, apply machine configurations, bootstrap the control plane, retrieve credentials, and create the kubeconfig that operators will eventually use to access the cluster. None of these steps are particularly difficult, but they are repetitive, and repetitive tasks are exactly the kind of things that bug me.

I wanted this deployment to be as close to a single-command experience as possible. If Terraform is already responsible for provisioning the infrastructure, why stop there? Why deploy the servers and then switch over to a collection of shell scripts and manual commands to finish the job? This is where the Talos Terraform Provider really shines. Instead of treating cluster initialization as a separate process, Terraform becomes responsible for the entire lifecycle of the deployment. Machine secrets are generated automatically, configurations are applied to the nodes, the control plane is bootstrapped, and both the `talosconfig` and `kubeconfig` files are created for me when everything is finished.

The result is a deployment process that feels almost boring and that's exactly what you want. I can deploy a cluster today, destroy it tomorrow, and rebuild it next week knowing the exact same sequence of events will occur every single time. No forgotten commands. No missing configuration files. No wondering which terminal history contained the bootstrap command I ran three months ago.

### Tailscale VPN

[Tailscale VPN Module](https://github.com/jgrove90/talos-foundry/tree/main/modules/vpn-tailscale) 

One thing I wanted to avoid in this deployment was exposing management interfaces directly to the internet. Sure, you can lock down the Kubernetes API to your home IP, open the Talos API to a trusted CIDR block, and call it a day. Plenty of people do exactly that. But the moment you start traveling, change ISPs, or need to administer the cluster from somewhere else, those firewall rules become an operational annoyance.

The traditional answer is usually some combination of VPN appliances, bastion hosts, jump boxes, or a dedicated management network. They work, but they also introduce more infrastructure to maintain. Now you're rotating credentials, managing SSH access, and troubleshooting why someone can't connect at 2 AM. Before long, you've built an entire platform just to reach your platform.

For this project, I wanted something simpler. Instead of exposing services publicly or maintaining a traditional VPN stack, I deployed a dedicated Tailscale subnet router inside AWS. Think of it as a secure bridge between my private VPC and authenticated devices on my Tailscale network. Once connected, I can reach the Kubernetes API, Talos management endpoints, and internal cluster services exactly as if I were sitting inside the VPC itself. The best part is that the entire setup is automated through Terraform. The EC2 instance, networking, security groups, and bootstrap configuration are all provisioned as code via a user-data script. During startup, the instance installs Tailscale, joins the tailnet, advertises the VPC routes, and immediately becomes a secure entry point into the environment.

The end result is a private cluster that remains private. No public Kubernetes API endpoint. No public Talos API endpoint. No bastion hosts. Just authenticated access through Tailscale whenever I need to administer the cluster. In my opinion, that's a much cleaner solution and maintanable solution.

## Looking Ahead: Karpenter, Cilium, and a More Cloud-Native Cluster

Traditionally, Kubernetes clusters are deployed with a fixed number of worker nodes. Maybe you start with three, realize you need five, manually update your infrastructure, and then eventually discover you're paying for idle compute most of the day. We've gotten pretty good at autoscaling pods, but many clusters are still surprisingly bad at autoscaling the infrastructure underneath them. This is where Karpenter comes in. Karpenter is a Kubernetes-native node provisioning system originally developed by AWS. Instead of managing Auto Scaling Groups and pre-defining worker node pools, Karpenter watches for unschedulable pods and provisions compute on demand. If a workload needs resources, Karpenter launches a node. When that node is no longer needed, Karpenter can terminate it automatically.

At the same time, I plan to replace the default networking stack with [Cilium](https://cilium.io/). While traditional Container Network Interfaces (CNIs) rely heavily on iptables and overlay networking, Cilium leverages eBPF directly within the Linux kernel to provide networking, observability, and security with significantly less overhead. Beyond performance improvements, Cilium unlocks powerful capabilities such as network policies, reverse proxy, service load balancing, traffic visibility, and cluster-wide observability through tools like [Hubble](https://github.com/cilium/hubble).

Throughout this project I've tried to embrace the idea that infrastructure should be disposable and automated. Karpenter extends that philosophy to worker nodes, and Cilium extends it to networking. Instead of managing fleets of servers and complex networking rules, the cluster becomes increasingly capable of managing itself. In the next post, I'll walk through deploying Karpenter on Talos Linux, integrating it with AWS, and replacing the default networking stack with Cilium to create a fully autoscaling, cloud-native Kubernetes platform.




