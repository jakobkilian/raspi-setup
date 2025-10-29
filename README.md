# Setting up a DietPi with Mixed Eth and Wi-Fi Access

## Considerations

This documents how I usually set up Raspberry Pis with [DietPi](https://dietpi.com/) to achieve:

- Fast booting, minimal Raspi with SSH and VNC connectivity
- Have Raspi connected to the internet via "a" Wi-Fi  (can change) for updates etc.
- Access Raspi fast and anywhere via ethernet (no Wi-Fi required)
- Seamless switching between Wi-Fi and ethernet by using the same hostname for both

Unfortunately, due to licensing restrictions, I cannot provide a bootable Raspberry Pi image. Instead, I will document my setup step by step. I may compromise on performance, security or standardisation here and there, but I prefer to use methods that I know work for me. Of course, choose a secure password and don't use the one from this tutorial.

I am working on a Mac; you'd need to adapt some steps if you're using a different operating system.

Contact me if you have any better ideas!

## Flash the OS

I love working with [DietPi](https://dietpi.com/). It's easy, fast and customisable – specifically for situations where you run it headless or need fast boot times. I reproduced this process in Oct 2025 with a Raspberry Pi 5 with 4GB of RAM using DietPi version 9.17 (Trixie) on a 32GB Sandisk Extreme microSD card.

Just Follow the [install](https://dietpi.com/docs/install/) docu: **Download**, **extract** and **flash**. For me the original [Pi Imager](https://www.raspberrypi.com/software/) worked better than [balenaEtcher](https://www.balena.io/etcher/). Select your Raspi and "own image" under OS and click "No" when asked about presets (we'll do that in the next step).

After flashing is completed reinsert the SD card in to computer again **before** booting the Raspi and:

- Open **dietpi.txt** through the file browser and set `AUTO_SETUP_NET_WIFI_ENABLED` from `=0` to `=1`.
- Open **dietpi-wifi.txt** and fill in credentials at `aWIFI_SSID` and `aWIFI_KEY`
- Put SD in Raspi and power up
- Find Raspi's IP e.g. [with this tutorial](https://raspberrytips.com/find-current-ip-raspberry-pi/). 
- Connect to the Raspi by calling `ssh root@192.168.0.XX` in the macOS terminal and using the default password `dietpi`
  - Tip: If you get a "WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED! " doing this, it means you had logged in to another device on the **same** IP/hostname. Do `ssh-keygen -R HOST` and replace HOST with IP or hostname to delete old keys.

- At first login a auto setup will run and you will be asked:
  - To change password for root, dietpi and software installation. Choose a secure password here, as our raspi will be accessible via Wi-Fi and ethernet! In this tutorial we'll call it <secure_pwd>
  - If you want to disable UART? → Yes
  - Navigate down to "install" → Enter → Enter to run the basic installation
  - You will be asked about sending anonymous usage data → your choice
  - You will end with a command line

## 1. Establish Access and Network

We want the raspy to act as router and DHCP server to allow direct ethernet connections (and bridging any Wi-Fi into it).

### 1 a) Update

It's good practice to run updates. If you followed this tutorial you can skip this, as dietpit updates at first boot.

```sh
sudo apt update && sudo apt upgrade -y
```

### 1 b) Install Packages

Let's install:

- `dnsmasq` and `avahi-daemon` for routing and network stuff
- `mosquitto` and `mosquitto-clients` for receiving MQTT (in later steps)

```sh
sudo apt install dnsmasq avahi-daemon mosquitto mosquitto-clients -y
```

### 1 c) Enable Services

```sh
sudo systemctl enable avahi-daemon
sudo systemctl enable mosquitto
sudo systemctl enable dnsmasq
```

"enable" will start them at the next boot up, "start" would do it right away. I prefer to do a clean reboot after setting everything, so no start needed here.

### 1 d) Config Network Settings

```sh
sudo nano /etc/dnsmasq.conf
```

Add at the end:

```
# act as ethernet router with DHCP and DNS
interface=eth0
dhcp-range=10.3.3.10,10.3.3.200,255.255.255.0,1h
dhcp-option=3,10.3.3.1
dhcp-option=6,10.3.3.1
dhcp-option=28,10.3.3.255
```

Save with `Ctrl + O` and leave with `Ctrl + X` 

### 1 e) Set Static IP for Raspi

````sh
sudo nano /etc/network/interfaces
````

**replace** (delete line with Ctrl +K) the whole ethernet paragraph with:

**Note:** `/etc/network/interfaces` is deprecated on some newer Raspberry Pi OS versions. DietPi supports it, but if you run into issues, check if your system uses `dhcpcd.conf` or another network manager.

```sh
allow-hotplug eth0
iface eth0 inet static
    address 10.3.3.1
    netmask 255.255.255.0
    gateway 10.3.3.1
```

This sets a static IP for the raspi at 10.3.3.1 and doesn't wait for external DHCP servers

### 1 f) Set Hostname

Setting hostname is easiest in the config menu of dietpi

```sh
dietpi-config
```
  - Go to: Security → Hostname → Set to `mypi` (or whatever you want)
  - You can also change your password here
  - Exit config

### 1 g) Ensure Avahi Start After Sth is Up

There can be problems, if avahi-daemon starts before the network is fully up (with the static ip assigned). This is why we wait with it until network is online:

````sh
sudo mkdir -p /etc/systemd/system/avahi-daemon.service.d
sudo nano /etc/systemd/system/avahi-daemon.service.d/override.conf
````

and then add this to the new file:

````sh
[Unit]
After=network-online.target
Wants=network-online.target
````

### 1 h) Reboot

````sh
sudo reboot
````

### 1 i) Test Setup

Now ethernet should be working. Check by doing: `ping 10.3.3.1`. 

Then exit, **disable Wi-Fi on your computer** and try again with: `ping mypi.local`. 

**Warning**: it can take some minutes (!) for a mDNS/Bonjour (.local) resolution to take place. So if it does not work for you, take a coffee and try later or simply stick with 10.3.3.1 for now...

mypi.local *could* also work via Wi-Fi but to my experience often fails because the router doesn't correctly handle the hostname via DNS. It has to work however with our direct ethernet connection to the Raspi, as we set it up to provide a DNS server itself.

You can now login with  `ssh dietpi@mypi.local`

If you have problems, connect the "old" way via Wi-Fi and do `ip addr show eth0 ` to check if sth is up and static IP is set. I can not really give you advice at that point, as I can't imagine what could go wrong.

##  2 Configure & Test MQTT

Let's now set up mosquitto and send a test message:

### Set up Mosquitto with password authentication

1. Create a password file and add a user (replace `<username>`):
  ```sh
  sudo mosquitto_passwd -c /etc/mosquitto/passwd <username>
  ```
  Enter a password when prompted.
    
2. Configure Mosquitto:
  ```sh
  sudo nano /etc/mosquitto/conf.d/remote.conf
  ```
And add this. This only allows MQTT connections with user and passwords for everyone and on port 1883

  ```
  # Mosquitto listener (password required)
  listener 1883 0.0.0.0
  allow_anonymous false
  password_file /etc/mosquitto/passwd
  ```

3. Set correct permissions:

  ```sh
  sudo chown mosquitto:mosquitto /etc/mosquitto/passwd
  sudo chmod 640 /etc/mosquitto/passwd
  ```

4. Restart Mosquitto:

  ```sh
  sudo systemctl restart mosquitto
  ```

### Test MQTT Receiving

On Raspberry Pi:

```sh
mosquitto_sub -h localhost -t test -u <username> -P <password>
```

On your remote computer you want to send MQTT messages. This can be done by using a GUI software like [MQTTX](https://mqttx.app/) or by the command line ([mosquitto](https://mosquitto.org/download/))

- Turn off your Wi-Fi on your remote Computer (to make sure eth works)
- Connect to the broker/server on your Raspi: `mqtt://mypi.local`
- Send a message with this content:
  - Topic: `test`
  - Payload: `Hello World!`

### test via Wi-Fi

- Unplug the ethernet cable and do the same thing agin. **This might fail depending on your router.** If DNS is not handled properly, you would alternatively need to use the (changing) IP address to connect via Wi-Fi.

## 3 Configure VNC Connection

Sometimes we want VNC access for easier maintenance. You can skip that point if you don't need it and only work in ssh anyways.

For VNC we do `dietpi-software` → search → type "vnc" → select tigerVNC with spacebar → Enter → install 

We will then be prompted to install a desktop environment (LXDE) and a browser (Firefox). The installation will take some minutes, you will be asked if you want to set up auto-login – select "yes" and set up auto-login with Desktop for user dietpi.

Now let's enable and start the service:

````sh
sudo systemctl enable vncserver
sudo systemctl start vncserver
````

Then connect with [RealVNC Viewer](https://www.realvnc.com/en/connect/download/viewer) or any other viewer to `mypi:local:1` or `10.3.3.1:1`. Note the `:1` which is the second screen, as Tiger VNC can not share the default HDMI screen (`:0`) if there is one.

You will be informed that there is no encryption (that is ok, we are local) and you have to enter your <secure_pwd>.







