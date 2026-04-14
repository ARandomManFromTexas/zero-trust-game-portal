# INFO, FAQ, AND TECHNICAL BREAKDOWN:
## This is more of a breakdown of everything you need to know, along with some problems and solutions.

--- 

--- 

## DISCLAIMER: 
The system architecture, network routing, hardware integration, and Linux environmental config were designed and tested by me from sources found online. Large language models (LLMs) were utilized strictly to generate the frontend Node.js/HTML boilerplate and assist in formatting the original Markdown document, allowing me to focus on the core pipeline and bug finding. 

Use at your own risk. This is not made for enterprise work; this is solely a hobbyist solution using pre-existing tech to get my PC running when I'm not at home. Feedback and iterations are needed and accepted, as I know this needs work. This is not highly secure—it is truly meant to stop unauthorized users and bots with the link. By using this script, you are using it at your own risk. I am not responsible if someone compromises it. Proceed with caution.

---

---

## INFO
First and foremost, I am a 15-year-old dev and this project was made as a way for me to use my PC when I am not at home. Iterations are greatly appreciated; I do not claim to know everything and feedback is accepted. The Linux setup on another device and the Cloudflare Zero Trust network setup were written by me. There is a common problem at the end, but that is more towards the setup guide. This will go into some other questions and problems I ran into, along with a technical breakdown to prove that I know what I'm talking about.

---
---
## FAQ
This will be updated as problems arise.

### Q: Do I need a Raspberry Pi?
### A: No, you do not. For this project, a Raspberry Pi is recommended because it doesn't use a lot of power and acts as an always-on infrastructure that will allow your website to keep running. But if you have an old PC or laptop, at the end of this guide there will be a tutorial on how to get Linux to run that will allow x86 to run the ARM infrastructure of the Pi.

### Q: Do I need an Ethernet cord?
### A: No, you do not. You can use Wi-Fi. The only step that I didn't change to show what to do for Wi-Fi is the JavaScript. You need to change `exec('etherwake -i eth0 ${MAC}'...` to `exec('etherwake -i wlan0 ${MAC}'...`.

### Q: Wake-on-LAN isn't working or it doesn't show up.
### A: The only real workaround I've found is getting a smart plug and going into the BIOS to turn "Restore on AC Power Loss" (or AC BACK/Recovery) to ON.

### Q: Can I use this on an Intel/AMD card?
### A: Yes, Sunshine supports both, but here be dragons. I have a 4060ti and have not experimented with either option, and I cannot provide support if it doesn't work.

### Q: Can I use this with multiple people/devices?
### A: Yes. If using one device, you will all have access to the same machine. As for multiple host devices, you can just go through the same pairing process as getting the first one set up.

### Q: Can I sell this guide or sell access to my website?
### A: While I physically cannot stop you, do note this is not secure. I am in no way, shape, or form a cybersecurity expert, but even I have noticed some ways in if someone wanted to.

### Q: Are you aware there is no auth system for the website? It can be bypassed by having the second subdomain leading to Moonlight.
### A: Yes, actually writing this here because I don't know how to bring it up. You could build your own JavaScript token to check if someone entered the first password. But if you just go with the Zero Trust route of requiring an email to get to the link, then it still achieves the same goal of allowing only you to get on.

### Q: Why is this going on your GitHub?
### A: Honestly, I do not expect anyone to read this, but I do want to be an embedded software engineer. While this project is not that, if I want to get on big tech companies' radars, projects like this will eventually help me get an internship. I've also heard from a lot of people in every aspect of the industry that just putting your stupid little Python, Java, or whatever projects out there is good just to show proof of skill.

---

---

# Technical Breakdown
This part I feel is going to be fairly boring, but we will be going step-by-step and explaining choices and what stuff means. If you don't care or already know, then feel free to skip this.

## Background info:
I came across Sunshine with Moonlight while trying to find a solution to using a PC on the go from a Chromebook, as some of my other dev projects were being limited by me not being able to work on them at school. This used to be an option offered by NVIDIA, but is now a separate project kept alive by a labor-of-love community.

---

---

## Phases 1-3:
###  Phase 1: Infrastructure & Discovery
Setting up the Raspberry Pi was straightforward. I used Fing for network scanning to identify the device's local IP and ensure it was visible on the subnet. To test the initial connection, I used Termius to SSH into the Pi. I made sure to test this over a WLAN-to-WLAN connection first to verify the routing was working before moving to a final Ethernet-to-WLAN configuration for better stability.
### Phase 2: Static Addressing & Security Trade-offs
I configured a Static IP to ensure the connection remains persistent. Without this, a router reboot could trigger a DHCP lease change, which would rotate the IP and break the backend code.
Security Note: I used sudo for quick configuration here. While this works for a home lab, in a production environment, I’d follow the Principle of Least Privilege. Using root-level access can be risky because a Zero-Day exploit could potentially compromise the entire OS rather than just a sandboxed environment.
### Phase3: Sunshine & GPU Accelerated Encoding
Stage 3 is where the heavy lifting happens. Sunshine is the backbone of the project. Its backend is impressive because it utilizes GPU-accelerated streaming (Hardware Encoding). This is much more efficient than software encoding. Since NVIDIA deprecated GameStream in favor of GeForce Now, Sunshine has become the gold standard for low-latency remote desktop access for the community.

---

---

## Phases 4
Phase 4: Wake-on-LAN
## Phase 4: Wake-on-LAN (WoL)
Wake-on-LAN is honestly one of the most technical and fun things I have gotten to learn. It really advanced my understanding of networking by helping me learn about the OSI model layers and how they operate in the real world.

### The Layer 2 Logic
WoL operates at **Layer 2 (the Data Link Layer)**. It uses what is called a **"Magic Packet"** containing the destination machine's unique **48-bit MAC address** repeated 16 times. Because the computer is asleep or in a "soft off" state, it doesn't have an active IP address, so the packet bypasses **Layer 3 (IP addressing)** and is broadcasted directly to the **NIC (Network Interface Card)**.

### Hardware & Power States
This relies heavily on **Layer 1 (Physical Layer)** connectivity. For this to work, you need a wired Ethernet connection so the motherboard can maintain "trickle power" to the NIC. This allows the network card to stay awake and "listen" for traffic even when the rest of the PC is powered down.

### The Trigger
Basically, the NIC listens to every broadcast on the local network. It ignores everything until it sees that specific frame containing its own MAC address 16 times. Once it recognizes that "key," it triggers the motherboard to boot the system. Even though these packets are usually sent using **UDP (Layer 4)**, the reason WoL works so well is that it functions at Layer 2, allowing us to reach a device that isn't even fully "on" yet.

---

---

## Phases 5
### Phases 5
This phase involves setting up the Raspberry Pi as a dedicated gateway for the stream. While some boilerplate for the web portal was generated with LLM assistance, the underlying infrastructure relies on several key Linux and networking concepts.

### Step 5A — Repository Synchronization & System Upgrades
First, we run sudo apt update the code we have on the pi with the online repository. Following that  we then just do sudo apt -y the y is just so we don't have to sit there clicking yes the sudo makes sure you have administrator privilege and apt is just the package tool to grab it

### Step 5B — Node.js Runtime Installation
We use curl to fetch the setup script from NodeSource. This script adds the specific repository needed for Node.js v20. Running node --version is a vital verification step to ensure the JavaScript runtime environment is correctly installed and ready to host our portal.

### Step 5C — Low-Level Packet Control (Etherwake)
We install etherwake to allow the Pi to generate and send **Magic Packets** at the Data Link layer. 
> **Note:** Due to specific hardware limitations or network configurations, standard WoL might not always trigger. In my personal testing, I utilized a **Smart Plug** workaround combined with a BIOS-level "AC Recovery" setting to ensure 100% boot reliability.

### Step 5D & 5E — Containerization with Docker
Bassically all this is is that we need to make sure everything can talk to each other cleanly without conflict on dependency so we do this with a Docker. When we make the docker for moonlight it gets its own enviorment to do its own thing without haveing to worry about other things in its way.
- The docker run command handles the heavy lifting: it sets the Restart Policy (unless-stopped), maps the necessary UDP ports for the stream, and defines the 1-to-1 NAT Host IP.
- We use docker ps to verify the container's status and ensure the process is active in the background.

### Step 5F — Pairing the Handshake (Sunshine/Moonlight)
As simply put as I can put this all step 5f is is that we paired the Pi to the Pc making sure they  could talk using a pin code to lockdown the connection which is what actually lets the stream show up on the website  securely.
---

---

# Phase 6: Building the Secure PortalPhase 6: Building the Secure Portal


### Step 6A and 6B: The Backend Logic

We use Node.js and Express to build a real server. The important part here is that the password check happens server-side. If you put your password check in the basic HTML code, anyone could just right-click, view source, and see your password. By doing it on the server, the user never sees the logic, they just get a yes or no. I also added a session secret which basically gives your browser a temporary ID card so you stay logged in for 8 hours without typing the password every 5 minutes.

### Step 6C: Security through Obscurity
I made a fake business page for a company called Summit Digital Solutions. It looks like a boring IT site so anyone who finds the link thinks they're in the wrong place. I added a hidden trigger where you have to click the logo 5 times in 3 seconds to get to the real login. It's not a bank-level fix, but it stops 99% of bots and random people from even knowing there is a login page. This has already worked as in my cloudflare  analytics page we have gotten some requests from Germany Netherlands India Finland Norway but no ip requests meaning it was more than likely bots that saw the  page was active and left after finding nothing.

### Step 6D and 6E: Persistence and Logging
I also added a logging system that writes every attempt to a file called access_log.txt. It records the IP address and if they succeeded or failed. This is great for visibility so I can see if someone is trying to brute force their way in. To make sure this stays running after I close my terminal, I used PM2. This is a process manager that keeps the script alive in the background and restarts it automatically if the Pi ever reboots.

---

---

# Step 7: The Cloudflare Tunnel
This part is simple but still very complex to anyone looking from the outside. Cloudflare is a great resource for hobbyist  like me as there free tiers offer a lot of protection and this way it creates a tunnel so that instead of opening a port on my router for anyone to DDoS me it instead goes through one of the biggest security teams on the planet. This also is simple for allowing me to edit my webpage instead of having to mess with anything else I just download Cloudflare add the IP and my domain and it edits it for me.

---

---

# Extra steps: Cloudflare and setting up a headless server
The headless server is a bit harder to fact check but the guide I got it from was credible so I trust it. The hardest part about this was getting ARM architecture what the pi uses onto a x86 device so we just went with a Linux distro Ubuntu that is great for servers and uses the same Linux as the pi.  As for Cloudflare this is fairly simple the zero trust network is amazingly simple and great to use. The backend support of even if you put the wrong email in it still goes through is a great security buffer as bots don't  try real emails to see which is correct they just get stuck on a login page and assume its the right one.
