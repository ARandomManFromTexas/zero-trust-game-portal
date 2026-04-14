# 🖥️ FULL SETUP GUIDE
## Stream Your Gaming PC From Any Browser — Raspberry Pi + Sunshine + Cloudflare

---

> **Read this first:** This guide goes in order. Do not skip phases.  
> Every command is copy-paste ready. When you see `YOUR_X` or 'yourdomain' in a command, replace it with your actual value.  
> If something breaks, stop and fix it before moving on.
> Use a strong password bitwarden is recommended but not necesary to store them.
> 
>if you dont have a Rasberry Pi and want to use another pc or laptop scroll down to the separate pc section written by me so not as nice as the stuff formatted by ai.

---

## 📋 WHAT YOU NEED BEFORE STARTING

- Gaming PC (Windows, with a NVIDIA card installed)
- Raspberry Pi 8GB with a fresh SD card (16GB minimum)
- SD card reader
- Two ethernet cables 
- Both Pi and PC plugged into your router via ethernet
- Cloudflare account with yourdomain domain already added
- A PC or laptop to flash the SD card and SSH from

---

---

# ⚡ PHASE 1 — Flash the Raspberry Pi

### Step 1 — Download Raspberry Pi Imager
Go to: **https://www.raspberrypi.com/software/**  
Download for Windows. Install and open it.

### Step 2 — Flash the SD card
1. Click **"Choose Device"** → pick your Pi model (Pi 4 or Pi 5)
2. Click **"Choose OS"** → **"Raspberry Pi OS (other)"** → **"Raspberry Pi OS Lite (64-bit)"**
3. Click **"Choose Storage"** → select your SD card
4. Click **"Next"** → when it asks about customization → click **"Edit Settings"**

### Step 3 — Configure settings in Imager
Fill these in:

| Setting | What to put |
|---|---|
| Hostname | `streampi` |
| Username | anything you want (e.g. `admin`) |
| Password | make it strong, WRITE IT DOWN |
| Enable SSH | ✅ Yes |
| Configure WiFi | ❌ Leave unchecked (we're using ethernet however if you're using wifi check and configure as needed) |

Click **Save** → **Yes** → it will flash. Takes a few minutes.

### Step 4 — Boot the Pi
- Put SD card in Pi
- Plug in ethernet cable to your router
- Plug in power
- Wait **2 full minutes** for first boot

### Step 5 — Find the Pi's IP address
Log into your router (usually `192.168.1.1` in your browser).  
Look for a device called `streampi` in the connected devices list.  
Write down its IP — it will look like `192.168.1.XX`

### Step 6 — SSH into the Pi
On your Windows PC, open **Command Prompt** and type:
```
ssh admin@192.168.1.XX
```
(replace `admin` with your username, and `192.168.1.XX` with your Pi's actual IP)

Type `yes` when it asks about authenticity. Enter your password. You're in.

---

---

# 📌 PHASE 2 — Set a Static IP on the Pi

This stops your Pi from getting a different IP every reboot which would break everything.

Run this command to open the network manager:
```bash
sudo nmtui
```

A blue menu appears. Use arrow keys:
1. Select **"Edit a connection"** → press Enter
2. Select **"Wired connection 1"** or **wifi name** if you're using wifi → press Enter
3. Arrow down to **"IPv4 CONFIGURATION"** → change `<Automatic>` to `<Manual>`
4. Arrow right to **`<Show>`** → press Enter
5. Fill in:
   - **Addresses:** type `192.168.1.50/24` (or pick any number 50-250 that isn't taken on your network)
   - **Gateway:** type your router IP (usually `192.168.1.1`)
   - **DNS servers:** type `8.8.8.8`
6. Arrow down to **`<OK>`** → press Enter
7. Press **Escape** → select **"Quit"**

Now restart networking:
```bash
sudo systemctl restart NetworkManager
```

Then reboot:
```bash
sudo reboot
```

Wait 30 seconds, then SSH back in — but this time use the new static IP you set:
```
ssh admin@192.168.1.50
```

---

---

# 🌞 PHASE 3 — Install Sunshine on Your Gaming PC

### Step 1 — Download Sunshine
On your Gaming PC, go to:  
**https://github.com/LizardByte/Sunshine/releases/latest**

Download the file called: `Sunshine-Windows-AMD64-installer.exe`

### Step 2 — Install it
- Run the installer
- Accept all defaults
- Let it install as a Windows Service (this means it starts automatically)
- Click Finish

### Step 3 — Open the Sunshine dashboard
Open your browser on the gaming PC and go to:
```
https://localhost:47990
```

Your browser will say **"Not Secure"** or show a warning. This is normal.  
Click **Advanced** → **Proceed to localhost**

### Step 4 — Create your login
It will ask you to set a username and password.  
**Make these strong and write them down.** This is your Sunshine admin login.

### Step 5 — Configure the encoder
1. Click **"Configuration"** at the top
2. Go to the **"Audio/Video"** tab
3. Find **"Encoder"** and set it to **NVENC** (that's your NVIDA card)
4. Click **Save**

Sunshine is now running in the background as a Windows service. It starts automatically every time your PC boots.

---

---

# 🖥️ PHASE 4 — Enable Wake-on-LAN on Your Gaming PC

This lets the Pi wake your PC remotely so it doesn't need to stay on 24/7.

### Step 4A — Enable WoL in BIOS
1. Restart your PC
2. Press **Delete** or **F2** (or **F10** depending on your motherboard) to enter BIOS
3. Look for **"Wake on LAN"** or **"Power On By PCI-E"** — it's usually under Power settings
4. Enable it
5. Save and exit (usually F10)

### Step 4B — Enable WoL in Windows
1. Right-click the Start button → **Device Manager**
2. Expand **"Network Adapters"**
3. Right-click your ethernet adapter → **Properties**
4. Go to **"Power Management"** tab
5. Check **"Allow this device to wake the computer"**
6. Go to **"Advanced"** tab
7. Find **"Wake on Magic Packet"** → set to **Enabled**
8. Click OK

### Step 4C — Find your PC's MAC address
Open Command Prompt on the gaming PC and type:
```
ipconfig /all
```
Look for your ethernet adapter. Write down the **Physical Address** — it looks like `A1-B2-C3-D4-E5-F6`

Also write down the **IPv4 Address** (e.g. `192.168.1.100`) — you'll need this later.

---

---

# 🍓 PHASE 5 — Set Up the Raspberry Pi Server

SSH back into the Pi first:
```
ssh admin@192.168.1.50
```

### Step 5A — Update the Pi
```bash
sudo apt update && sudo apt upgrade -y
```
This takes a few minutes. Let it run.

### Step 5B — Install Node.js
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install nodejs -y
```

Verify it worked:
```bash
node --version
```
Should show `v20.x.x`

### Step 5C — Install etherwake (for Wake-on-LAN)
```bash
sudo apt install etherwake -y
```

Allow it to run without sudo (needed for our web app):
```bash
sudo setcap cap_net_raw+ep /usr/sbin/etherwake
```

### Step 5D — Install Docker (for moonlight-web-stream)
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
```

Log out and back in so Docker permissions apply:
```bash
exit
```
SSH back in:
```
ssh admin@192.168.1.50
```

### Step 5E — Run moonlight-web-stream
This is the thing that lets your browser connect to Sunshine.

Run this command (replace `192.168.1.50` with your actual Pi IP):
```bash
docker run -d \
  --name moonlight-web \
  --restart unless-stopped \
  -p 8080:8080 \
  -p 40000-40100:40000-40100/udp \
  -e WEBRTC_NAT_1TO1_HOST=192.168.1.50 \
  mrcreativ3001/moonlight-web-stream:latest
```

Check it's running:
```bash
docker ps
```
You should see `moonlight-web` in the list.

### Step 5F — Pair moonlight-web-stream with Sunshine

On any browser **at home** (not on public wifi, has to be on your home network), go to:
```
http://192.168.1.50:8080
```

1. Create an account — **first user becomes admin.** Set a strong username and password.
2. Click the **"+"** button to add a PC
3. For the address, enter your gaming PC's local IP (e.g. `192.168.1.100`)
4. Click the **pair button** (chain link icon)
5. A **PIN code** appears on screen — don't close this

Now on your Gaming PC:
- Open `https://localhost:47990` in browser (Sunshine dashboard)
- Click **"PIN"** in the top menu
- Enter the PIN from moonlight-web-stream → click Send

**Test it:** click your PC in the moonlight-web-stream interface. You should see your desktop streaming in the browser! If that works, you're golden.

---

---

# 🔒 PHASE 6 — Build the Secure Portal (Fake Page + Server-Side Login + IP Logging)

This builds your fake landing page, hidden login, and Node.js server that:
- Checks your password on the server (not in browser code anyone can read)
- Logs every login attempt with IP and timestamp
- Never exposes the real streaming URL until you're authenticated

### Step 6A — Create the project folder
```bash
mkdir ~/portal
cd ~/portal
npm init -y
npm install express express-session fs
```

### Step 6B — Create the main server file

```bash
nano server.js
```

Paste this entire block:

```javascript
const express = require('express');
const session = require('express-session');
const fs = require('fs');
const path = require('path');

const app = express();

// ============================================================
// CHANGE THESE:
const PASSWORD = 'YourStrongPasswordHere';   // your login password
const SESSION_SECRET = 'pick-a-random-phrase-here-abc123';
const STREAM_URL = 'http://192.168.1.50:8080'; // moonlight-web-stream local URL
// ============================================================

app.use(express.urlencoded({ extended: true }));
app.use(express.json());
app.use(session({
  secret: SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: { maxAge: 8 * 60 * 60 * 1000 } // session lasts 8 hours
}));

// Serve static files from /public
app.use(express.static(path.join(__dirname, 'public')));

// Log function — writes to access_log.txt
function logAttempt(ip, success, note = '') {
  const timestamp = new Date().toISOString();
  const status = success ? 'SUCCESS' : 'FAILED ';
  const line = `[${timestamp}] ${status} | IP: ${ip.padEnd(20)} | ${note}\n`;
  fs.appendFileSync(path.join(__dirname, 'access_log.txt'), line);
  console.log(line.trim());
}

// Main page — serves the fake landing page
app.get('/', (req, res) => {
  res.sendFile(path.join(__dirname, 'public', 'index.html'));
});

// Login page — served at /go (hidden path)
app.get('/go', (req, res) => {
  if (req.session.authenticated) {
    return res.redirect('/stream');
  }
  res.sendFile(path.join(__dirname, 'public', 'login.html'));
});

// Handle login form submission
app.post('/auth', (req, res) => {
  const { password } = req.body;
  const ip = req.headers['x-forwarded-for'] || req.socket.remoteAddress;

  if (password === PASSWORD) {
    req.session.authenticated = true;
    logAttempt(ip, true, 'Login successful');
    return res.redirect('/stream');
  } else {
    logAttempt(ip, false, `Wrong password attempt`);
    return res.redirect('/go?error=1');
  }
});

// Stream redirect — only accessible after login
app.get('/stream', (req, res) => {
  if (!req.session.authenticated) {
    return res.redirect('/go');
  }
  res.redirect(STREAM_URL);
});

// Wake on LAN endpoint
app.post('/wake', (req, res) => {
  if (!req.session.authenticated) {
    return res.status(403).send('Unauthorized');
  }
  const MAC = 'A1:B2:C3:D4:E5:F6'; // REPLACE with your gaming PC's MAC address
  const { exec } = require('child_process');
  exec(`etherwake -i eth0 ${MAC}`, (err) => {
    if (err) {
      console.error('WoL error:', err);
      return res.json({ success: false });
    }
    res.json({ success: true });
  });
});

// View logs (authenticated only)
app.get('/logs', (req, res) => {
  if (!req.session.authenticated) {
    return res.redirect('/go');
  }
  try {
    const logs = fs.readFileSync(path.join(__dirname, 'access_log.txt'), 'utf8');
    res.set('Content-Type', 'text/plain');
    res.send(logs);
  } catch {
    res.send('No logs yet.');
  }
});

app.listen(3000, () => {
  console.log('Portal running on port 3000');
});
```

Press **Ctrl+X** → **Y** → **Enter** to save.

### Step 6C — Create the public folder and pages

```bash
mkdir public
```

**Create the fake landing page:**
```bash
nano public/index.html
```

Paste this — it's a boring fake business page. The hidden trigger is clicking the logo 5 times:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Summit Digital Solutions</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { font-family: Arial, sans-serif; background: #f4f4f4; color: #333; }
    header { background: #2c3e50; color: white; padding: 20px 40px; display: flex; align-items: center; justify-content: space-between; }
    header h1 { font-size: 1.4em; font-weight: 600; cursor: default; user-select: none; }
    nav a { color: #aaa; text-decoration: none; margin-left: 20px; font-size: 0.9em; }
    .hero { background: #3498db; color: white; text-align: center; padding: 80px 20px; }
    .hero h2 { font-size: 2em; margin-bottom: 10px; }
    .hero p { font-size: 1.1em; opacity: 0.85; }
    .services { display: flex; justify-content: center; gap: 30px; padding: 60px 40px; flex-wrap: wrap; }
    .card { background: white; border-radius: 8px; padding: 30px; width: 250px; box-shadow: 0 2px 8px rgba(0,0,0,0.08); }
    .card h3 { margin-bottom: 10px; color: #2c3e50; }
    .card p { font-size: 0.9em; color: #666; line-height: 1.6; }
    footer { background: #2c3e50; color: #aaa; text-align: center; padding: 20px; font-size: 0.85em; }
  </style>
</head>
<body>
  <header>
    <h1 id="logo">Summit Digital Solutions</h1>
    <nav>
      <a href="#">Services</a>
      <a href="#">About</a>
      <a href="#">Contact</a>
    </nav>
  </header>

  <div class="hero">
    <h2>Professional IT Services</h2>
    <p>Reliable infrastructure solutions for modern businesses.</p>
  </div>

  <div class="services">
    <div class="card">
      <h3>Network Management</h3>
      <p>We design and maintain secure, scalable business networks tailored to your needs.</p>
    </div>
    <div class="card">
      <h3>Cloud Solutions</h3>
      <p>Seamless migration and management of cloud infrastructure across all major platforms.</p>
    </div>
    <div class="card">
      <h3>IT Support</h3>
      <p>24/7 support desk keeping your systems running with minimal downtime.</p>
    </div>
  </div>

  <footer>
    &copy; 2025 Summit Digital Solutions. All rights reserved.
  </footer>

  <script>
    // Secret: click the logo 5 times to go to login
    let clicks = 0;
    let timer;
    document.getElementById('logo').addEventListener('click', () => {
      clicks++;
      clearTimeout(timer);
      timer = setTimeout(() => { clicks = 0; }, 3000);
      if (clicks >= 5) {
        clicks = 0;
        window.location.href = '/go';
      }
    });
  </script>
</body>
</html>
```

Press **Ctrl+X** → **Y** → **Enter**

**Create the login page:**
```bash
nano public/login.html
```

Paste this:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Login</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body {
      font-family: Arial, sans-serif;
      background: #1a1a2e;
      display: flex;
      align-items: center;
      justify-content: center;
      min-height: 100vh;
    }
    .box {
      background: #16213e;
      border-radius: 12px;
      padding: 40px;
      width: 340px;
      box-shadow: 0 8px 32px rgba(0,0,0,0.4);
    }
    h2 {
      color: #e0e0e0;
      margin-bottom: 6px;
      font-size: 1.4em;
    }
    .sub {
      color: #888;
      font-size: 0.85em;
      margin-bottom: 28px;
    }
    label {
      display: block;
      color: #aaa;
      font-size: 0.8em;
      margin-bottom: 6px;
      text-transform: uppercase;
      letter-spacing: 0.05em;
    }
    input {
      width: 100%;
      padding: 12px;
      border-radius: 6px;
      border: 1px solid #2a2a4a;
      background: #0f3460;
      color: white;
      font-size: 1em;
      margin-bottom: 20px;
      outline: none;
    }
    input:focus { border-color: #e94560; }
    button {
      width: 100%;
      padding: 13px;
      background: #e94560;
      color: white;
      font-size: 1em;
      font-weight: bold;
      border: none;
      border-radius: 6px;
      cursor: pointer;
      transition: background 0.2s;
    }
    button:hover { background: #c73652; }
    .error {
      background: rgba(233, 69, 96, 0.15);
      border: 1px solid #e94560;
      color: #e94560;
      padding: 10px;
      border-radius: 6px;
      margin-bottom: 16px;
      font-size: 0.85em;
      display: none;
    }
    .error.show { display: block; }
  </style>
</head>
<body>
  <div class="box">
    <h2>Secure Access</h2>
    <p class="sub">Enter your credentials to continue</p>

    <div class="error" id="err">Incorrect password. Try again.</div>

    <form method="POST" action="/auth">
      <label>Password</label>
      <input type="password" name="password" placeholder="••••••••••" autofocus required />
      <button type="submit">Sign In</button>
    </form>
  </div>

  <script>
    // Show error message if ?error=1 in URL
    if (window.location.search.includes('error=1')) {
      document.getElementById('err').classList.add('show');
    }
  </script>
</body>
</html>
```

Press **Ctrl+X** → **Y** → **Enter**

### Step 6D — Edit the server.js with your real values

Open server.js again:
```bash
nano server.js
```

Find these lines near the top and change them:
```javascript
const PASSWORD = 'YourStrongPasswordHere';      // ← change this
const SESSION_SECRET = 'pick-a-random-phrase-here-abc123'; // ← change this to any random words
const STREAM_URL = 'http://192.168.1.50:8080';  // ← should already be right
```

Also find this line and replace the MAC address:
```javascript
const MAC = 'A1:B2:C3:D4:E5:F6'; // ← put your gaming PC's MAC address here
```

Save with **Ctrl+X** → **Y** → **Enter**

### Step 6E — Test it locally first

```bash
node server.js
```

On your home computer browser go to: `http://192.168.1.50:3000`

You should see the fake business page. Click the title ("Summit Digital Solutions") **5 times** — it should jump to the login page. Enter your password — it should redirect to moonlight-web-stream.

Press **Ctrl+C** in the Pi terminal to stop it when done testing.

### Step 6F — Make the server run automatically forever

Install PM2 (keeps Node running even after reboot):
```bash
sudo npm install -g pm2
pm2 start server.js --name portal
pm2 startup
```

PM2 will output a command for you to run (copy and paste it — it looks like `sudo env PATH=...`). Run it.

Then:
```bash
pm2 save
```

Now your portal server starts automatically every time the Pi boots.

---

---

# ☁️ PHASE 7 — Connect Cloudflare Zero Trust Tunnel

### Step 7A — Install cloudflared on the Pi

```bash
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main' | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt update && sudo apt install cloudflared -y
```

### Step 7B — Create the tunnel in Cloudflare dashboard

1. Go to **dash.cloudflare.com** → **Zero Trust** → **Networks** → **Tunnels**
2. Click **"Create a tunnel"**
3. Choose **Cloudflared** → Next
4. Name it `streampi` → Save
5. On the next screen choose **"Debian"** (same as Pi OS)
6. Copy the install command — it looks like:
   ```
   sudo cloudflared service install eyJhIjoiXXXXXXXX...
   ```
7. **Run that command on your Pi**

It registers automatically as a service and starts.

### Step 7C — Add public hostnames

In the Cloudflare dashboard, still in your tunnel settings, click **"Public Hostnames"** tab → **"Add a public hostname"**

Add **two** hostnames:

**Hostname 1 — Main portal:**
| Field | Value |
|---|---|
| Subdomain | `stream` |
| Domain | `your domain` |
| Service Type | `HTTP` |
| URL | `localhost:3000` |

**Hostname 2 — moonlight-web-stream (for after login):**
| Field | Value |
|---|---|
| Subdomain | `app` |
| Domain | `your domain` |
| Service Type | `HTTP` |
| URL | `localhost:8080` |

Save both.

### Step 7D — Update STREAM_URL in server.js

Now that you have a public URL for moonlight-web-stream, update it:

```bash
nano ~/portal/server.js
```

Change:
```javascript
const STREAM_URL = 'http://192.168.1.50:8080';
```
To:
```javascript
const STREAM_URL = 'https://app.yourdomain';
```

Save, then restart:
```bash
pm2 restart portal
```

---

---

# ✅ FINAL TEST

1. Go to `https://yourdomain` in any browser
2. You see a boring business website
3. Click the title **5 times**
4. Login page appears
5. Enter your password
6. You land on moonlight-web-stream
7. Click your gaming PC
8. Stream starts 🎉

---

---

# 🌙 WAKE-ON-LAN (so your PC doesn't stay on 24/7)

After logging in, you can wake your PC by visiting:
```
https://yourdomain/wake
```

Or you can add a "Wake PC" button in the login page. When you click it, it sends a magic packet to your gaming PC. Wait 30 seconds, then connect.

To check who has tried to log into your portal:
```
https://yourdomain/logs
```
(You have to be logged in first)

---

---

# How to set up a separate PC.
*(Do this instead of Phase 1 if you are using an old PC or laptop instead of a Raspberry Pi)*
### **Disclaimer**: I don't currently have a spare laptop to test this specific "Old PC" route. The Linux commands are standard and should work flawlessly, but if you run into any quirks, feel free to open an Issue and ask me!
## What you need: 
- a 8 gig usb
- the pc being converted into a headless server
- another pc to download rufus


### Step 1 - Download Ubuntu Server
 1. Go to **https://rufus.ie/** and download Rufus.
 
 2. Click **"Download Ubuntu Server"** (Get the latest LTS version, like 24.04 LTS).

 3. this will download an `.iso` file.
### Step 2 - Download Rufus and flash your usb.
 1. plug an empty USB flash drive into your current pc
 2. Go to **https://rufus.ie/** and download Rufus.
 3. Open Rufus.
   - **Device:** Select your USB drive.
   - **Boot Selection:** Click "SELECT" and find the Ubuntu Server `.iso` file you just downloaded.
 4. Click **START** and wait for it to finish.

### Step 3 - Boot the Old PC
 1. Plug the flashed USB drive into the old PC you want to use as a server.
 2. Plug the old PC directly into your router via ethernet.
 3. Turn the PC on and immediately start tapping the **Boot Menu Key** (Usually `F12`, `F8`, `F2`, or `DEL` depending on the computer brand).
 4. Select your USB drive from the boot menu.
### Step 4 — Run the Installer
The Ubuntu Server installer is entirely text-based. Use your arrow keys and the `Enter` key to navigate.
 1. Choose your language and keyboard layout.
 2. **Network Connections:** It should automatically detect your ethernet connection and get an IP address. Hit "Done".
 3. **Storage Configuration:** Leave it on the default "Use an entire disk" to wipe the old PC and install Linux.
 4. **Profile Setup:** Fill this in carefully!
    - Your name: `admin` (or whatever you want)
   - Your server's name: `streampi` (Keep it the same as the Pi guide!)
   - Pick a username: `ubuntu`
   - Choose a strong password.
### Step 5 — CRITICAL: Enable SSH
On the "SSH Setup" screen, **YOU MUST CHECK THE BOX that says "Install OpenSSH server".** *(Press the Spacebar to check the box).* If you skip this, you will not be able to control the server remotely!
### Step 6 — Finish and Reboot
 1. Let the installation finish. It will take 5 to 10 minutes. 
 2. When it says "Reboot Now", select it.
3. Pull the USB drive out when the screen goes black.
### Step 7 - Connect to your own server
 Go back to your main PC. Open Command Prompt or PowerShell and connect to your new server just like you would with the Pi: '' bash 
 ssh ubuntu@YOUR_SERVER_IP
 

if you dont know your IP go to your router and find it.

---

---

# Setting up extra security with cloudflare
the good part about using zero trust for our login is that cloudflare has built in security thatll protect against bots and can add a email check if you would like only a specific email to be able to route  traffic. This does not fix the being able to open either link issue but if they dont have your email all theyll see is just a email check. Completly optional but a good idea overall.

### Step 1 - Add the Application
1. Log into the Cloudflare Zero Trust Dashboard (`https://one.dash.cloudflare.com/`).
2. On the left menu, go to **Access** > **Applications**.
3. Click **Add an application** and select **Self-Hosted**.
4. **Application Configuration:** Name it "Stream Portal" and select your subdomain from the dropdown (e.g., `app.yourdomain.com`). if you dont see subdomain 'click add public host name'
5. Scroll down to Access policies and click  create new policy.

### Step 2 - Creating a new policy
1.  Click **policy names** and select a name for it. Then change it to allow and same as application time out.
2. Go to **Add rules** include should already be there if not click add include
3. Once in **Add rules** click selector and select email from the dropdown menu then for the value add what ever email you want to work. and click save.

### Step 3 - Finishing your Application
1.  Go back to your application in the **Access policy** box click select existing policy
2. Then select  whatever you named your policy from the list.
3. Go down click next feel free to customize what the email  page will look like if not click next agian
4. Finaly click save wait a second and go to your second domain  for moonlight login and youll see what ever you decided on for your email page.

### **Important note youll see enter a email page no matter what email you put this will still say accept the code the only email thatll get a code is the one you put in your policy. Cloudflare has everyone see the enter a code page so that bots do try a million emails to find yours instead the bot sees a code page and moves on.**

---

---

# 🔒 YOUR SECURITY LAYERS

| Layer | What it does | Who it stops |
|---|---|---|
| Decoy  homepage | Decoy | Random people who stumble onto the URL |
| 5-click secret trigger | Obscurity | Automated scanners find nothing |
| Server-side password | Real auth | Anyone trying to log in |
| Session cookie | Stays logged in 8hrs | You don't re-login constantly |
| moonlight-web-stream password | Second gate | Extra layer on the stream itself |
| Sunshine password | Third gate | Nobody gets to the stream host |
| IP logging | Visibility | You can see who's trying |
| No open router ports | Network security | Your home network is never directly exposed |

---

# 🐛 COMMON PROBLEMS & FIXES

| Problem | Fix |
|---|---|
| Can't SSH into Pi | Make sure ethernet is plugged in. Find IP in router device list. |
| `docker ps` shows nothing | Run the Docker command again. Check for typos. |
| Moonlight-web-stream won't pair | Make sure both Pi and gaming PC are on same network. Check Sunshine is running. |
| Portal shows error on `/go` | Make sure `node server.js` is running, or PM2 is active (`pm2 list`) |
| Stream has lag | Normal on public WiFi. On 5G hotspot it'll be smooth. |
| WoL doesn't work | Double-check BIOS setting is enabled. Check MAC address has no typos. |
| Can't reach yourdomain.com | Check cloudflared is running: `sudo systemctl status cloudflared` |

---

*Last updated for this setup: April 2026*
