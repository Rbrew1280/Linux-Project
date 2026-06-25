Phase 1: Base Operating System Setup
Install Ubuntu Desktop LTS on the Chromebox.
During the installation steps, ensure you check the box for "Log in automatically."
Once booted to the desktop, open Settings → Power and disable Screen Blanking / Automatic Suspend so the screen never goes to sleep.
Phase 2: System Packages & Firewall Setup
Open a terminal (Ctrl + Alt + T) and paste this block to update the system, open the network ports for AirPlay mirroring, and install the Docker engine:
Bash
# 1. Update the operating system packages
sudo apt update && sudo apt full-upgrade -y

# 2. Open standard firewall ports for UxPlay mirroring & mDNS discovery
sudo ufw allow 5353/udp
sudo ufw allow 7000:7002/tcp
sudo ufw allow 7000:7002/udp
sudo ufw reload

# 3. Install AirPlay streaming dependencies and the Avahi network daemon
sudo apt-get install -y ca-certificates curl avahi-daemon gstreamer1.0-plugins-bad gstreamer1.0-libav gstreamer1.0-vaapi

# 4. Install the official Docker Engine and Compose plugins
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 5. Enable Docker to start on system boot
sudo systemctl enable docker.service
sudo systemctl enable containerd.service

# 6. Authorize your user account to manage Docker containers without sudo
sudo usermod -aG docker $USER

⚠️ CRITICAL: You must reboot the Chromebox (sudo reboot) right now before proceeding to Phase 3, or the user permissions won't apply and the following steps will throw permission errors.
Phase 3: Deploy the Signage Stack (Anthias)
Once rebooted, open your terminal back up and build the local digital signage container engine using our customized, hardcoded settings.
Create the screenly folder structure:

mkdir -p ~/screenly && cd ~/screenly




Open a new configuration file:

nano docker-compose.yml




Paste this exact configuration block, which uses the correct x86 images, fixes the memory issues, and strips out the broken Raspberry Pi driver dependencies:

version: '3'

services:
  redis:
    image: ghcr.io/screenly/anthias-redis:latest-x86
    restart: always
    mem_limit: 512m

  anthias-server:
    image: ghcr.io/screenly/anthias-server:latest-x86
    restart: always
    depends_on:
      - redis
    environment:
      - HOME=/data
      - LISTEN=0.0.0.0
      - CELERY_BROKER_URL=redis://redis:6379/0
      - CELERY_RESULT_BACKEND=redis://redis:6379/0
    volumes:
      - ./data:/data
    ports:
      - "80:80"
    mem_limit: 512m

  anthias-celery:
    image: ghcr.io/screenly/anthias-server:latest-x86
    restart: always
    depends_on:
      - redis
      - anthias-server
    command: celery -A anthias_server.celery_tasks.celery worker -B -n worker@anthias --loglevel=info
    environment:
      - HOME=/data
      - CELERY_BROKER_URL=redis://redis:6379/0
      - CELERY_RESULT_BACKEND=redis://redis:6379/0
    volumes:
      - ./data:/data
    mem_limit: 512m

  anthias-viewer:
    image: ghcr.io/screenly/anthias-viewer:latest-x86
    restart: always
    shm_size: 128m
    depends_on:
      - anthias-server
    environment:
      - DISPLAY=:0
      - WLR_DRM_NO_ATOMIC=1
    volumes:
      - /tmp/.X11-unix:/tmp/.X11-unix
    mem_limit: 512m

Press Ctrl + O, then Enter to save, and Ctrl + X to exit the text editor.
Launch the containers in background mode:

docker compose up -d




Phase 4: Configure the Firefox Proxy Whitelist
Because the network filters block standard web looping, we must whitelist local playback.
Open Firefox manually.
Click the Menu (three lines) in the top right corner → Settings.
Scroll down to the very bottom of the General tab and click Settings... under Network Settings.
Find the text box labeled "No proxy for" and add this exact string to it:

localhost, 127.0.0.1, 192.168.0.0/16, 10.0.0.0/8




Click OK and close Firefox.
Phase 5: Automate Screen Mirroring (UxPlay Service)
Create a system service so the AirPlay receiver starts automatically at boot on pinned, unblocked ports.
Open a terminal file creator:

sudo nano /etc/systemd/system/uxplay.service




Paste this configuration block (replace yourusername with the actual name of your Chromebox Ubuntu user profile):

[Unit]
Description=UxPlay AirPlay Receiver
After=network.target avahi-daemon.service docker.service

[Service]
Type=simple
User=yourusername
Environment=DISPLAY=:0
ExecStart=/usr/bin/uxplay -p 7000 -fs -vsync no
Restart=on-failure

[Install]
WantedBy=graphical.target




Save and close (Ctrl + O, Enter, Ctrl + X).
Register and enable the background hook:

sudo systemctl daemon-reload
sudo systemctl enable uxplay.service
sudo systemctl start uxplay.service




Phase 6: Automate the Fullscreen Kiosk on Boot
Finally, create the startup rule that launches Firefox directly into fullscreen mode after waiting for Docker to finish loading.
Make sure the hidden user autostart folder exists:

mkdir -p ~/.config/autostart




Create the startup config file:

nano ~/.config/autostart/anthias-kiosk.desktop




Paste this desktop metadata inside:

[Desktop Entry]
Type=Application
Exec=sh -c "sleep 10 && firefox --kiosk http://127.0.0.1"
Hidden=false
NoDisplay=false
X-GNOME-Autostart-enabled=true
Name=Anthias Signage Kiosk
Comment=Launches local digital signage player after a 10s delay




Save and close (Ctrl + O, Enter, Ctrl + X).
Give the startup shortcut executable rights:

chmod +x ~/.config/autostart/anthias-kiosk.desktop




Final Verification
Give the system a full reboot: systemctl reboot -i
The Chromebox will log in autonomously, sit on the desktop for 10 seconds while Docker initializes the layout, and then snap open into a beautiful full-screen digital signage loop. You can now connect to it via AirPlay at any time or manage your slides by pulling up the device's IP address on your laptop!

