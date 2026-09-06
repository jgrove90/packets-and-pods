---
title: "From Gaming PC to AI Server: Building a Hybrid Workstation"
date: 2026-08-30T14:00:00-06:00
lastmod: 2026-08-30T14:00:00-06:00
draft: false
summary: "How I turned my increasingly underutilized gaming PC into a hybrid AI server using llama.cpp, llmfit, and systemd without giving up its day job as a desktop."
featuredImage: "/images/2-from-gaming-pc-to-ai-server/llama-cpp.webp"
images: ["/images/2-from-gaming-pc-to-ai-server/llama-cpp.webp"]

tags: ["Kubernetes", "Linux", "AI"]
categories: ["AI Infrastructure"]

lightgallery: true
---

# A Second Life for My Desktop

I believe open source models are the future, despite what the AI overlords and their fear-mongering would have us believe.

Over the past few years, my desktop hasn't exactly been earning its keep. I haven't been gaming nearly as much, which means my relatively capable Radeon 6800 XT has spent a lot of time doing very little.

At the same time, AI infrastructure has become increasingly interesting to me. So rather than let a perfectly good GPU become an expensive paperweight, I decided to give the machine a second job: running local AI workloads.

The goal isn't to turn my desktop into a dedicated server. I still want to use it as a desktop especially with the possibility of Grand Theft Auto VI eventually making its way to PC (can my 6800 XT even run it? That's a problem for future me). Instead, I'm building a hybrid setup where the machine can pull double duty as an AI inference server when I'm not gaming or otherwise using it.

It should also complement my Kubernetes homelab nicely and, more importantly, give me a place to experiment with local AI infrastructure and inference tuning without spending a fortune on cloud GPUs.

I don't have anything particularly crazy running here, but here's the hardware I'm working with: 

![System resource usage while running the AI server](/images/2-from-gaming-pc-to-ai-server/system-resources.webp)

# Hybrid Server Approach With systemd

The goal here is a machine that could switch between roles without a bunch of manual intervention. When I'm using the desktop, it should behave like a desktop. When I'm not, it should be available as an AI inference server.

For this, I decided to use ```systemd```. It gives me a simple way to manage the inference server as a service, allowing me to start it when I want to run AI workloads and stop it when I need the GPU back for gaming or other desktop workloads. Otherwise, the machine can sit there quietly acting as an AI server.

Once I started working through the problem, I realized there were several pieces I needed to solve:

1. Power management — I don't want the machine consuming power just to sit around waiting for an AI request. If I'm not using it, it should be able to sleep.
2. Wake-on-LAN — If the machine is asleep, I need a way to wake it up when I actually want to use the AI server.
3. Model selection — Find a model that makes sense for the hardware I actually have, rather than blindly downloading the biggest model I can find.
4. Inference endpoint — Expose the model through an endpoint that fits into the rest of my homelab and can be consumed by other services.
5. Network access — Configure the firewall to allow the necessary traffic without opening up more of the machine than I need to.

None of these problems are particularly difficult on their own. The interesting part is putting them together into something that behaves more like infrastructure while still being my everyday desktop.

# The Implementation 

systemd services inbound!

### Fixing My Sleep

I've had a weird problem with this computer for the last several Linux kernel updates: it wouldn't stay asleep.

I'd suspend the machine, the screen would go dark, and a few seconds later it would wake right back up. This started sometime around four or five kernel updates ago, and even updating the BIOS to the latest version didn't solve it.

Since power management is pretty important for this project, I needed to figure out what was waking the machine up. Unfortunately, this turned into one of those troubleshooting sessions involving far too much trial and error.

Eventually, I narrowed it down to an ACPI wakeup source called GPP0.

ACPI, or the Advanced Configuration and Power Interface, is the firmware interface Linux uses to communicate with things like power states, devices, and wakeup events. The kernel exposes some of these wakeup sources through /proc/acpi/wakeup.

GPP0 is an ACPI-defined PCIe/General Purpose Port wakeup source. On my system, it was being allowed to wake the machine immediately after suspend. The exact device behind GPP0 is hardware and firmware dependent, but in my case disabling it was enough to let the machine actually stay asleep.

The fix ended up being surprisingly simple. I created a systemd service that disables the GPP0 wakeup source during boot:

```
[Unit]
Description=Disable GPP0 ACPI Wakeup
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c "grep -q 'GPP0.*enabled' /proc/acpi/wakeup && echo GPP0 > /proc/acpi/wakeup || true"

[Install]
WantedBy=multi-user.target

```
Writing the name of a wakeup source to /proc/acpi/wakeup toggles its state. The service first checks whether GPP0 is currently enabled and only disables it if necessary.

This is one of those fixes that feels a little ridiculous until you realize you've spent an hour watching a computer suspend for approximately four seconds before waking itself back up.

### Configuring Wake-On-LAN

Now that the sleep issue was fixed, I needed a way to remotely wake the desktop when I wanted to use it for AI.

The first step was in the BIOS. I enabled Wake-on-LAN and disabled ErP (Energy-related Products). ErP can cut power to the network hardware during sleep, which defeats the whole purpose of Wake-on-LAN. Most modern motherboards support this, and it's a surprisingly useful feature.

Unfortunately, enabling Wake-on-LAN in the BIOS wasn't enough. Linux also needs to be configured to listen for Wake-on-LAN packets.

I used ```ethtool``` to enable it on my Ethernet interface:

```
sudo ethtool -s eno1 wol g
```
The problem is that this setting doesn't persist across reboots. So, naturally, I made systemd do it for me:

```
[Unit]
Description=Enable Wake-on-LAN
After=network-pre.target
Before=network.target

[Service]
Type=oneshot
ExecStart=/usr/bin/ethtool -s eno1 wol g
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Now the BIOS keeps the network interface available during sleep, and systemd enables Wake-on-LAN every time Linux boots.

Now that begs the question, how do I wake up the desktop?  With a little magic of course! I installed a small utility called ```wakeonlan``` that sends a Wake-on-LAN magic packet to a network interface using its MAC address. If the machine is asleep and everything is configured correctly, receiving that packet is enough to wake it back up.

```
wakeonlan <MAC_ADDRESS>
```

That's it. One packet, and the desktop wakes up.

# Choosing an Inference Engine

With the hardware sorted out, the next question was what I was actually going to use to run the models.

The two obvious choices were vLLM and llama.cpp.

vLLM is designed primarily around high-performance model serving. It makes a lot of sense when you're running larger models, handling multiple concurrent requests, and trying to maximize throughput. It also provides an OpenAI-compatible API, making it a good fit for production-style inference workloads.

llama.cpp takes a different approach. It's lightweight, runs on both CPUs and GPUs, has strong support for quantized models, and includes its own HTTP server with an OpenAI-compatible API. It also supports things like continuous batching and parallel requests, so it isn't limited to running one prompt at a time.

In a larger GPU cluster where throughput is the primary concern, I'd probably reach for vLLM. For this situation, however, llama.cpp made more sense.

I'm working with a single desktop GPU, I want to experiment with different quantized models, and I care more about getting the most out of the hardware I already own than maximizing requests per second.

The next step is tuning llama.cpp specifically for this machine. Things like GPU offloading, context size, batch size, and the model's quantization can have a significant impact on memory usage and inference performance. Rather than assuming the default settings are optimal, I'll benchmark different configurations and find the sweet spot for my hardware in a future blog post. 

## Choosing the model

Choosing a model can be overwhelming. How do you know what will actually run on your hardware? There are thousands of models on Hugging Face, an open, maybe not that open model registry, with the recent purchase by Nvidia. Additionally, each model comes in multiple sizes and quantizations.

Being a fan of terminal user interfaces (TUIs), I found a much easier way to answer that question: ```llmfit```.

![llmfit](/images/2-from-gaming-pc-to-ai-server/llmfit.webp)


```llmfit``` detects your hardware and estimates which models will fit, what quantization makes sense, and how they should perform. Instead of downloading a 20GB model and finding out the hard way that my GPU isn't particularly interested in running it, I can see what makes sense before downloading anything. Using the TUI, I was able to browse models based on my actual hardware and narrow down the options pretty quickly.

For this setup, I settled on Qwen3-14B Q6_K in GGUF format.GGUF is a model file format designed for local inference. It packages the information needed to run a model—including its weights, tokenizer, and configuration—into a single file. It is also the format used by llama.cpp.

The Q6_K part refers to the quantization. In simple terms, quantization reduces the precision used to store the model's weights, which significantly reduces memory requirements compared with running the model at full precision. The tradeoff is that lower quantization can reduce model quality, while higher quantization uses more memory. Q6_K sits toward the higher-quality end of the common quantization options, giving me a good balance between memory usage and model quality for the hardware I have.

The result is a model that's large enough to be useful without completely consuming the machine's resources. 

# Ready for Launch

With the model selected, it was finally time to launch the server.

Thankfully, this part is pretty straightforward. llama.cpp includes llama-server, which exposes the model over HTTP and provides an OpenAI-compatible API. I wrapped it in a systemd service so it starts automatically, restarts if it fails, and behaves like any other service on the machine.
```
[Unit]
Description=llama.cpp AI Server
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=jgrove
WorkingDirectory=/home/jgrove

ExecStart=/usr/bin/llama-server \
    -m /home/jgrove/.cache/llmfit/models/Qwen3-14B-Q6_0.gguf \
    --host 0.0.0.0 \
    --port 9931 \
    -ngl all \
    -c 16384

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```
Lets check its running:
![llmfit](/images/2-from-gaming-pc-to-ai-server/systemd.webp)

There are a few important options here. -ngl all offloads as much of the model as possible to the GPU, while -c 16384 gives the model a 16K context window. Binding to 0.0.0.0 allows other machines on my network to reach the API instead of limiting it to localhost.

Since the server is listening on the network, I also needed to open port 9931 in Uncomplicated Firewall (UFW). Rather than exposing it broadly, I limited access to my local subnet:

```
 sudo ufw allow from 192.168.1.0/24 to any port 9931 proto tcp comment 'llama.cpp LAN API'
```

At this point, anything on my LAN can reach the llama.cpp API directly using the desktop's IP address.That works, but typing an IP address and port isn't exactly how the rest of my homelab operates. My reverse proxy already runs inside Kubernetes, handles TLS, and provides internal DNS endpoints, so I wanted the AI server to fit into that same infrastructure. The only complication is that llama.cpp isn't actually running inside Kubernetes.

To bridge that gap, I created a Kubernetes Service and EndpointSlice that point to the desktop's LAN address:

```
apiVersion: v1
kind: Service
metadata:
  name: llama-server
  namespace: utils
spec:
  ports:
    - name: http
      port: 9931
      targetPort: 9931
---
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: llama-server
  namespace: utils
  labels:
    kubernetes.io/service-name: llama-server
addressType: IPv4
ports:
  - name: http
    protocol: TCP
    port: 9931
endpoints:
  - addresses:
      - 192.168.1.123

```

From there, I can use my existing NGINX Ingress setup just like I would for an application actually running in the cluster:

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: llama-ingress
  namespace: utils
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

spec:
  ingressClassName: nginx

  tls:
    - hosts:
        - elodin.alarlab.dev
      secretName: wildcard-alarlab-tls

  rules:
    - host: elodin.alarlab.dev
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: llama-server
                port:
                  number: 9931
```
Now instead of remembering something like 192.168.1.123:9931, I can access the server through https://elodin.alarlab.dev on my local network.

Kubernetes handles the reverse proxy and TLS, while the actual inference workload stays exactly where I want it: directly on the desktop and its GPU.

It's a slightly unconventional setup, but that's also what I like about it. The desktop isn't part of the Kubernetes cluster, yet it can still take advantage of the networking and ingress infrastructure I've already built around it.

# What's Next?

What started as an attempt to give an underutilized GPU something useful to do turned into a pretty fun little infrastructure project. I now have a desktop that can still be my desktop when I want it to be, but can also wake up, run local AI workloads, and integrate with the rest of my homelab when I don't need it to be a Desktop.

There are definitely things I want to improve. I'd like to experiment more with llama.cpp tuning, benchmark different models and quantizations, and set one or more Hermes agents. For now, though, I'm pretty happy with the result. Instead of buying another server or paying for cloud GPU time, I managed to turn hardware I already owned into another useful piece of my homelab. 

