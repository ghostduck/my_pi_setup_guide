# The quick cheatsheet

## System-side

The contents are probably copy from somewhere else from the web or the steps

To update packages and reboot:

```bash
# Update package list
sudo apt update

# Install them
sudo apt full-upgrade # -y  # if you dare or for automation # otherwise manually enter y to continue

# Remove unwanted packages
sudo apt autoremove # -y  # if you dare or for automation # otherwise manually enter y to continue

sudo reboot
```

To really shutdown the server:

```bash
sudo poweroff
```

## Pi-hole

The URL is `http://<ip of server>/admin` or `http://pi.hole/admin` if Pi-Hole is already your DNS server.

To reset Web Interface password:

```bash
pihole -a -p
```

Backup settings:

```bash
# Feel free to change the file name
pihole -a -t pihole_settings.tar.gz
```

Seems we can only import the settings from the Web UI?

Update block list:

```bash
pihole -g
```

Update Pi-hole

```bash
pihole -up
```

## Pi-specific

Open the helper

```bash
sudo raspi-config
```
