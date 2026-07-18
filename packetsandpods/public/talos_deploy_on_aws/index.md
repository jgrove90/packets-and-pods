# Terraform, Talos, and the Art of the Disposable Server


<!--more-->

## In the land of AWS EKS, why deploy Talos Linux?

I’ve been running Talos Linux in my homelab for over two years now. Some people might say it is overkill and part of me understands that sentimnt and another part of me thinks those nay sayer's don't understand how to operate Kubernetes and that it actually provides alot of convience. That opinion deserves a whole other blog post. Talos makes managing and deploying Kubernetes a breeze by representing a fundamental shift in how we think about infrastructure. Instead of the traditional Kubernetes deployment on top of an operating system, Talos made Kubernetes the operating system. After all, if Kubernetes is the operating system of the cloud, why are we still managing Linux distros underneath it? Sidero Labs has essentially stripped away everything unnecessary, leaving only what’s required to run containers reliably. To be honest, I’m surprised it isn’t the defacto Kubernetes deployment everywhere. Maybe there are edge cases I haven’t hit yet. Having deployed Talos on proxmox in my homelab I thought it would be a good exercise in what configuraion is needed to get Talos up and running in AWS. Before I dig into the AWS infrastructure side of things I want to discuss what I like about Talos, primarlily the immutability/ephermal nature and security of the operating system.

### Immutability: From Pets to Cattle

In the DevOps space, we talk a lot about "Pets vs. Cattle." In the past, you treat your servers like pets: you name them, you nurse them back to health when they're sick, and you manage their individual updates and user configurations. But with Talos, we’re strictly in the "Cattle" business. These servers are ephemeral. You aren't a parent; you’re a butcher. Need a security update? You don’t patch the running server you send it to the slaughterhouse and roll out a new one. Need to add a new configuration? Spin up a new instance and tear down the old one. By automating this, you ensure your environment is consistent, repeatable, and most importantly completely disposable.This is where Talos really shines. It’s immutable by design. For example, I need the i915 extension to pass my onboard GPU through to Kubernetes for Plex transcoding. You can’t just "install" that once the system is live. Instead, Talos gives you the Talos Factory to bake those extensions directly into the image before you deploy. Need a change? You don’t log in and tweak settings—you just update your image and redeploy. There is something very satisfying about destorying architecutre and having it spun back up in the same state it was before it was destoryed. 

### Security: No SSH, 

 

## The Architecture

### Compute

### Network

### Security

### Talos Bootstrap

### Tailscale VPN

## App

### Kustomize

## Conclusion




