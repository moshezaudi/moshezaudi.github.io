---
layout: distill
title: The Ultimate Plex Guide - Overseerr, Sonarr, Jackett, and Radarr
date: 2024-03-10
description: Plex guide to Overseerr, Sonarr, Jackett, and Radarr for seamless media management and streaming perfection.
tags: plex, overseerr, sonarr, jackett, radarr
categories: technology, entertainment
pretty_table: true
toc:
  - name: Introduction
  - name: Important Disclaimer
  - name: Security Context
  - name: Prerequisites
  - name: Installation
  - name: Mounting SMB Share
  - name: Configuring Applications
  - name: Conclusions
  - name: Credits
---

### Introduction

The combination of Overseerr, Sonarr, Jackett, and Radarr alongside Plex creates an ultimate media management and streaming system that offers several benefits:

1. **Automation:** Sonarr is a tool for automatically downloading TV shows as soon as they become available, ensuring that your media library is always up to date. Radarr performs a similar function for movies. This automation saves time and effort by eliminating the need to search for and download content manually.

2. **Content Management:** Jackett acts as a "middleman" between Sonarr/Radarr and various torrent sites, allowing these tools to search for and download content from multiple sources. This increases the range of available content and makes it easier to find specific shows or movies.

3. **Media Server:** Plex serves as the media server that hosts and streams your content to various devices, offering a user-friendly interface and seamless playback experience. It can organize your media library, provide metadata, and support a wide range of formats, ensuring compatibility with different devices.

4. **Customization:** Overseerr is a tool that integrates with Plex to provide a user-friendly request system for TV shows and movies. Users can request specific content that is not available in the library, and Overseerr automates the process of searching for and adding requested content, enhancing the user experience and satisfaction.

---

### Important Disclaimer

The information provided is for educational purposes only. I do not take responsibility for any actions taken based on the information provided.

---

### Security Context

If any of the applications you installed following this guide can be accessed outside of your internal network, or if you worry about your little hacker brother, make sure to change all default passwords and review the security settings for each application.

---

### Prerequisites

- 6 x Ubuntu 22.04 LXC containers (64 Bit)
- ready-to-use SMB Share (for storage)

---

### Installation

First, we will need to install each application in its respective container.

There are several reasons to leverage containers for this task, primarily to ensure that each application remains isolated and operates independently, rather than running everything on a single, resource-heavy server.

Installation list:

- [plex](https://plex.tv)
- [overseerr](https://github.com/sct/overseerr)
- [sonarr](https://github.com/Sonarr/Sonarr)
- [qbittorrent](https://github.com/qbittorrent/qbittorrent)
- [jackett](https://github.com/Jackett/Jackett)
- [radarr](https://github.com/Radarr/Radarr)

Below are installation commands for each application; after reviewing the documentation for all of these programs, these are the commands that will work with a copy and paste and get you ready to go on your first try (and wow, it took me a while).

Remember to run each installation in a separate container.

---

### Plex Installation

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl gnupg2 -y
echo deb https://downloads.plex.tv/repo/deb public main | sudo tee /etc/apt/sources.list.d/plexmediaserver.list
curl https://downloads.plex.tv/plex-keys/PlexSign.key | sudo apt-key add -
sudo apt update
sudo apt install plexmediaserver
```

---

### Overseerr Installation

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install docker.io -y
docker run -d \
  --name overseerr \
  -e LOG_LEVEL=debug \
  -e TZ=Asia/Tokyo \
  -e PORT=5055 \
  -p 5055:5055 \
  -v /path/to/appdata/config:/app/config \
  --restart unless-stopped \
  sctx/overseerr
```

---

### Sonarr Installation

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl -y
curl -o- https://raw.githubusercontent.com/Sonarr/Sonarr/develop/distribution/debian/install.sh | sudo bash
```

---

### qBittorrent Installation

Before we set up qbittorrent-nox to run as a background service, it's advisable to run it once so that we can get some configuration out of the way such as the legal disclaimer.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install qbittorrent-nox supervisor -y
```

Then run `qbittorrent-nox`. It will prompt you to accept the legal disclaimer.

Quit the running qbittorrent-nox process by pressing `Ctrl-c` on your keyboard:

```bash
^CCatching signal: SIGINT
Exiting cleanly
```

Finally, continue with setting up the service:

```bash
sudo systemctl start supervisor
sudo systemctl enable supervisor
cat << EOF | sudo tee /etc/supervisor/conf.d/qbittorrent.conf > /dev/null
[program:qbittorrent]
command=qbittorrent-nox
autostart=true
autorestart=true
stderr_logfile=/var/log/qbittorrent.err.log
stdout_logfile=/var/log/qbittorrent.out.log
EOF
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start qbittorrent
```

---

### Jackett Installation

```bash
sudo apt update && sudo apt upgrade -y
sudo adduser --gecos "" --disabled-password jackett
cd /opt && wget https://github.com/Jackett/Jackett/releases/download/v0.21.1981/Jackett.Binaries.LinuxAMDx64.tar.gz
tar -vxf Jackett.Binaries.LinuxAMDx64.tar.gz
chown jackett:jackett -R "/opt/Jackett"
sudo ./Jackett/install_service_systemd.sh
```

---

### Radarr Installation

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl sqlite3 supervisor -y
sudo systemctl start supervisor
sudo systemctl enable supervisor
wget --content-disposition 'http://radarr.servarr.com/v1/update/master/updatefile?os=linux&runtime=netcore&arch=x64'
tar -xvzf Radarr*.linux*.tar.gz
sudo mv Radarr /opt/
cat << EOF | sudo tee /etc/supervisor/conf.d/radarr.conf > /dev/null
[program:radarr]
command=/opt/Radarr/Radarr -nobrowser -data=/var/lib/radarr/
autostart=true
autorestart=true
stderr_logfile=/var/log/radarr.err.log
stdout_logfile=/var/log/radarr.out.log
EOF
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start radarr
rm Radarr*.linux*.tar.gz
```

---

### Ports Overview

Now that all our applications are up and running, here is an overview of the default ports:

| :------------: | :-------: |
| Plex | 32400 |
| Overseerr | 5055 |
| Sonarr | 8989 |
| Jackett | 9117 |
| Radarr | 7878 |
| qBittorrent | 8080 |

---

### Mounting SMB Share

We will now proceed by mounting the SMB share to the Plex container.

Copy the following line, replace the placeholders with your information, and then paste it into the `/etc/fstab` file.

```bash
//<smb_ip>/<smb_path> /mnt/plex cifs username=<smb_username>,password=<smb_password>,uid=<uid>,gid=<gid>,file_mode=0664,dir_mode=0775 0 0
```

<small>Note: You can retrieve your `uid` and `gid` via `id -u` and `id -g`.</small>

Now, we will execute the following commands to create the mount directory and mount the SMB share:

```bash
sudo mkdir /mnt/plex
sudo mount -a
```

We will follow the same steps for Sonarr, Radarr, and qBittorrent containers.

---

### Configuring Applications

We will now proceed with configuring each application through the browser.

---

### Jackett Setup

<ol>
    <li>Open a browser and visit <code>http://&lt;jackket_ip:9117&gt;</code></li>
    <li>Click on "Add Indexers"</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/jackett-screenshot-06.jpg" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>
        Search for any indexers you want and add them. I will add the YTS indexer for movies and the EZTV indexer for TV shows.
    </li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/jackett-screenshot-03.jpg" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>Click on the settings icon and then click "Okay."</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/jackett-screenshot-02.jpg" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>After adding the desired indexers, Jackett should look like this:</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/jackett-screenshot-01.jpg" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
</ol>

---

### qBittorrent Setup

<ol>
    <li>Open a web browser and enter <code>http://&lt;qbittorrent_ip&gt;:8080</code>.</li>
    <li>Log in using the default credentials: username <code>admin</code> and password <code>adminadmin</code>.</li>
    <li>Click on "Tools" and then select "Options".</li>
    <li>Ensure that separate folders are set up for downloading and saving files. In the Downloads tab, designate <code>/mnt/plex/downloading</code> as the location for incomplete torrents and <code>/mnt/plex/torrents</code> for completed torrents. Make sure to enable the option to keep incomplete torrents in a different path. Save the changes.</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/qBittorrent-screenshot-02.png" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>To optimize the performance of our qBittorent application and manage downloads more efficiently, we will now activate torrent queueing and seeding. Navigate to the BitTorrent tab and enable these features. This will help prevent overloading the container with downloads and ensure that rate limits are handled effectively.</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/qBittorrent-screenshot-01.png" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>Ensure you have saved all changes before proceeding to the next step.</li>
</ol>

---

### Plex Setup

<ol>
    <li>Open a web browser and enter <code>http://&lt;plex_ip&gt;:32400/web</code>.</li>
    <li>Navigate through the initial basic welcome and account setup.</li>
    <li>Click on the settings icon and then under "Manage" select "Libraries".</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/plex-screenshot-05.png" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>Click on "Add Library".</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/plex-screenshot-04.png" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>Choose "Movies" and click "Next".</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/plex-screenshot-03.png" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>Enter the path <code>/mnt/plex/torrents/</code> and click "Add".</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/plex-screenshot-02.png" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>Finally, click on "Add Library" to finish adding the library.</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/plex-screenshot-01.png" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>Repeat the same steps for adding TV shows to your Plex library.</li>
</ol>

---

### Sonarr Setup

<ol>
    <li>Open a web browser and enter <code>http://&lt;sonarr_ip&gt;:8989</code>.</li>
    <li>Navigate to the settings, then select "Indexers" and click on the plus icon to add a new indexer.</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/sonarr-screenshot-01.png" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>Select "Torznab" and continue.</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/sonarr-screenshot-03.png" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>Now, retrieve the API Key and “Torznab Feed” URL for the EZTV Indexer from Jackett.</li>
    <li>Go ahead and copy these details, then paste them into the corresponding fields as shown below.</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/jackett-screenshot-07.png" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/sonarr-screenshot-02.png" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>Next, select all the TV categories or only the specific ones we want. Be sure to click "Test" before saving.</li>
    <li>Navigate to the settings, then select "Download Clients" and click on the plus icon to add a new download client.</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/sonarr-screenshot-04.png" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>Select "qBittorrent" and continue.</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/sonarr-screenshot-05.png" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>In the form that has been opened, enter the qBittorrent IP address in the "Host" field, and then provide the username, password, and categories accordingly. (The default username and password are <code>admin</code> and <code>adminadmin</code>.) Be sure to click "Test" before saving.</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/sonarr-screenshot-06.png" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>Be sure to click "Test" before saving.</li>
    <li>Next, Navigate to the settings, then select "Connect" and click on the plus icon to add a new connection.</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/sonarr-screenshot-07.png" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>Select "Plex Media Server".</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/sonarr-screenshot-08.png" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>Fill in the host with your Plex IP address, and click authenticate with Plex TV.</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/sonarr-screenshot-09.png" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>Be sure to click "Test" before saving.</li>
    <li>For the final step within Sonarr, navigate to the settings, click “Media Management” check the box “Rename Episodes” and then click on “Add Root Folder”.</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/sonarr-screenshot-10.png" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>Add the path <code>/mnt/plex/torrents/</code> and click ok.</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/sonarr-screenshot-11.png" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>Make sure everything is saved and Sonarr is ready to go.</li>
</ol>

---

### Radarr Setup

We will set up Radarr in the same way we set up Sonarr, as they are essentially identical, with the only distinction being that Radarr is used for movies and Sonarr is used for TV shows.

---

### Overseerr Setup

<ol>
    <li>Open a web browser and enter <code>http://&lt;overseerr_ip&gt;:5055</code>.</li>
    <li>Log in with your Plex account.</li>
    <li>Configure Plex settings by adding the Plex IP address and save the changes.</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/overseerr-screenshot-07.png" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>In the Plex Libraries section, enable the "Movies" and "TV Shows" options, then click on "Continue".</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/overseerr-screenshot-06.png" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>Proceed to click on "Add Sonarr Server".</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/overseerr-screenshot-05.png" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>Select the "Default Server" option, and fill in the required fields for "Server Name", "IP Address", and "API Key".</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/overseerr-screenshot-04.png" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>The Sonarr API key can be found under "Settings" > "General". Simply copy the key.</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/sonarr-screenshot-12.png" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>Click the test button to load existing profiles.</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/overseerr-screenshot-02.png" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>Customize the "Quality Profile" as needed, designate <code>/mnt/plex/torrents</code> as the root folder, select "Deprecated" for the Language Profile (as it is mandatory), and click on "Add server".</li>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/overseerr-screenshot-01.png" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <div class="row mt-3">
        <div class="col-sm mt-3 mt-md-0">
            {% include figure.liquid loading="eager" path="assets/img/posts/1/overseerr-screenshot-08.png" class="img-fluid rounded z-depth-1" zoomable=true %}
        </div>
    </div>
    <li>Repeat the same process for adding the Radarr server, and you'll be all set to start using the service.</li>
</ol>

---

### Conclusions

After trying out and getting to know each of these applications, and carefully going over their documentations, I can say that they make it easy to watch stuff and keep your library organized, setting it up may present some challenges, but the effort is worthwhile! Plus, you can adjust the settings to make it even better. Each application has a bunch of useful features, and when you use them all together, it's the best way to manage and stream media.

---

### Credits

[Home media server with plex, sonarr, radarr, qbitorrent and overseerr](https://dev.to/rafaelmagalhaes/home-media-server-with-plex-sonarr-radarr-qbitorrent-and-overseerr-2a84) (Rafael Magalhaes, 2023)
