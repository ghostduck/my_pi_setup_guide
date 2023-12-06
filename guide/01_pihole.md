# Pi-hole setup

I really want a pi-hole for my home DNS.

## Install Pi-hole

The steps following the offical github repo - I prefer method 2 (Manually download the installer and run) as at 9 Jul 2023.

```bash
wget -O basic-install.sh https://install.pi-hole.net # download and check it
sudo bash basic-install.sh # Mostly just choose yes and press enter and choose eth0 for the network interface
```

Then you need to make your devices to use this new DNS. You can achieve this in various ways (from very easy to very troublesome).

- Change DNS server in router
- Disable DHCP in router, then use DHCP in pi-hole
- Change DNS for each devices

The router from my ISP does not allow me to change DNS so I have to use the second way...

Inside the Web UI, add some ad list for Pi-hole to block.

This website has great amount of lists to copy and paste. <https://firebog.net/>

## Use unbound for Pi-hole

Reference: <https://docs.pi-hole.net/guides/dns/unbound/>

About these section: Some deeper knowledge about DNS system is required.

Knowing that DNS can resolve a hostname to IP address alone is not enough. We need to know (at least) DNS Recursive Resolver/DNS recursor and Authoritative name server in other to appreciate this setup. (Root nameserver and TLD nameserver seems shadowed by the final destination - Authoritative name server in this "try to understand" section)

This setup is not necessary but I would prefer this for better privacy.

Normally we end our setup at the above stage and the pi-hole would be totally fine and be able to serve its purpose.

However, there are some privacy concerns about the public upstream DNS (DNS Recursive Resolver) servers pi-hole is using, like Google's 8.8.8.8 and Cloudflare. What if they became evil or got DNS-poisoned or compromised or they are just gathering your querying history?

(By the way I assume those Root, TLD and Authoritative servers are not compromised... Not sure if this is wrong or not)

So we setup unbound in our Pi-hole server and make our server become the only DNS Recursive Resolver to be used in my home network.

### Install and configure unbound

```bash
sudo apt install unbound
```

After installing unbound with above command, it should fail to start as port 53 is already used. We need to configure it.

```bash
sudo vi /etc/unbound/unbound.conf.d/pi-hole.conf  # create new file in dir owned by root
```

Contents of `pi-hole.conf` copy from pi-hole offical guide:

Also copying highlight from the guide about this conifg:

- Listen only for queries from the local Pi-hole installation (on port 5335)
- Listen for both UDP and TCP requests
- Verify DNSSEC signatures, discarding BOGUS domains
- Apply a few security and privacy tricks

```txt
server:
    # If no logfile is specified, syslog is used
    # logfile: "/var/log/unbound/unbound.log"
    verbosity: 0

    interface: 127.0.0.1
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes

    # May be set to yes if you have IPv6 connectivity
    do-ip6: no

    # You want to leave this to no unless you have *native* IPv6. With 6to4 and
    # Terredo tunnels your web browser should favor IPv4 for the same reasons
    prefer-ip6: no

    # Use this only when you downloaded the list of primary root servers!
    # If you use the default dns-root-data package, unbound will find it automatically
    #root-hints: "/var/lib/unbound/root.hints"

    # Trust glue only if it is within the server's authority
    harden-glue: yes

    # Require DNSSEC data for trust-anchored zones, if such data is absent, the zone becomes BOGUS
    harden-dnssec-stripped: yes

    # Don't use Capitalization randomization as it known to cause DNSSEC issues sometimes
    # see https://discourse.pi-hole.net/t/unbound-stubby-or-dnscrypt-proxy/9378 for further details
    use-caps-for-id: no

    # Reduce EDNS reassembly buffer size.
    # IP fragmentation is unreliable on the Internet today, and can cause
    # transmission failures when large DNS messages are sent via UDP. Even
    # when fragmentation does work, it may not be secure; it is theoretically
    # possible to spoof parts of a fragmented DNS message, without easy
    # detection at the receiving end. Recently, there was an excellent study
    # >>> Defragmenting DNS - Determining the optimal maximum UDP response size for DNS <<<
    # by Axel Koolhaas, and Tjeerd Slokker (https://indico.dns-oarc.net/event/36/contributions/776/)
    # in collaboration with NLnet Labs explored DNS using real world data from the
    # the RIPE Atlas probes and the researchers suggested different values for
    # IPv4 and IPv6 and in different scenarios. They advise that servers should
    # be configured to limit DNS messages sent over UDP to a size that will not
    # trigger fragmentation on typical network links. DNS servers can switch
    # from UDP to TCP when a DNS response is too big to fit in this limited
    # buffer size. This value has also been suggested in DNS Flag Day 2020.
    edns-buffer-size: 1232

    # Perform prefetching of close to expired message cache entries
    # This only applies to domains that have been frequently queried
    prefetch: yes

    # One thread should be sufficient, can be increased on beefy machines. In reality for most users running on small networks or on a single machine, it should be unnecessary to seek performance enhancement by increasing num-threads above 1.
    num-threads: 1

    # Ensure kernel buffer is large enough to not lose messages in traffic spikes
    so-rcvbuf: 1m

    # Ensure privacy of local IP ranges
    private-address: 192.168.0.0/16
    private-address: 169.254.0.0/16
    private-address: 172.16.0.0/12
    private-address: 10.0.0.0/8
    private-address: fd00::/8
    private-address: fe80::/10
```

Then restart the unbound service. Finally check if it works or not.

```bash
sudo service unbound restart # same as # sudo systemctl restart unbound
systemctl status unbound  # should be enabled and active

# Testing
$ dig pi-hole.net @127.0.0.1 -p 5335

; <<>> DiG 9.16.42-Raspbian <<>> pi-hole.net @127.0.0.1 -p 5335
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 33152
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;pi-hole.net.			IN	A

;; ANSWER SECTION:
pi-hole.net.		300	IN	A	3.18.136.52

;; Query time: 59 msec
;; SERVER: 127.0.0.1#5335(127.0.0.1)
;; WHEN: Wed Jul 19 03:08:17 BST 2023
;; MSG SIZE  rcvd: 56
```

Recommendation from the official guide - Add limit to FTL (dnsmasq)

```bash
sudo bash -c "echo 'edns-packet-max=1232' >> /etc/dnsmasq.d/99-edns.conf"
```

More checking: DNSSEC validation.

Mostly copy from the guide but the failed example does not work so I copied another example from Cloudflare blog.

> The first command should give a status report of SERVFAIL and no IP address. The second should give NOERROR plus an IP address.

```bash
$ dig fail01.dnssec.works @127.0.0.1 -p 5335  # original example but timeout instead of SERVFAIL

; <<>> DiG 9.16.42-Raspbian <<>> fail01.dnssec.works @127.0.0.1 -p 5335
;; global options: +cmd
;; connection timed out; no servers could be reached

# Copy and edit from Cloudflare blog https://blog.cloudflare.com/unwrap-the-servfail/
# The line we want to check is:
# ;; ->>HEADER<<- opcode: QUERY, status: SERVFAIL, id: 18873
$ dig dnssec-failed.org @127.0.0.1 -p 5335

; <<>> DiG 9.16.42-Raspbian <<>> dnssec-failed.org @127.0.0.1 -p 5335
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: SERVFAIL, id: 18873
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;dnssec-failed.org.		IN	A

;; Query time: 1251 msec
;; SERVER: 127.0.0.1#5335(127.0.0.1)
;; WHEN: Wed Jul 19 03:21:19 BST 2023
;; MSG SIZE  rcvd: 46

# Note the "status: SERVFAIL"

$ dig dnssec.works @127.0.0.1 -p 5335

; <<>> DiG 9.16.42-Raspbian <<>> dnssec.works @127.0.0.1 -p 5335
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 40357
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;dnssec.works.			IN	A

;; ANSWER SECTION:
dnssec.works.		3600	IN	A	5.45.107.88

;; Query time: 31 msec
;; SERVER: 127.0.0.1#5335(127.0.0.1)
;; WHEN: Wed Jul 19 03:16:38 BST 2023
;; MSG SIZE  rcvd: 57

# Note the "status: NOERROR"
```

If all good, update the DNS server settings in pi-hole so that only the local DNS recursor will be used. (In the web interface, set `127.0.0.1#5335` in Upstream DNS Servers)

Still we are not done yet...

> Debian Bullseye+ releases auto-install a package called openresolv with a certain configuration that will cause unexpected behaviour for pihole and unbound. (as at 19 Jul 2023)

So we have to workaround it...

(Update on 4 Dec 2023) I reinstall the Pi using the latest Lite image. The Debian version becomes 12 and the codename is bookworm. I cannot find the `unbound-resolvconf.service`, `/etc/resolvconf.conf` and the `/etc/unbound/unbound.conf.d/resolvconf_resolvers.conf`... I guess I can skip these steps.

```bash
# confirm Debian version is bullseye+ or not
$ lsb_release -a

No LSB modules are available.
Distributor ID:	Raspbian
Description:	Raspbian GNU/Linux 11 (bullseye)
Release:	11
Codename:	bullseye

# About the service we have to stop
# Step 1 - Disable the Service
$ systemctl is-active unbound-resolvconf.service  # output active

$ sudo systemctl disable --now unbound-resolvconf.service  # Then disable it

# Step 2 - Disable the file resolvconf_resolvers.conf
# Comment out the unbound_conf= line in /etc/resolvconf.conf
# And then remove the resolvconf_resolvers.conf
$ sudo sed -Ei 's/^unbound_conf=/#unbound_conf=/' /etc/resolvconf.conf
$ sudo rm /etc/unbound/unbound.conf.d/resolvconf_resolvers.conf

# Finally restart service
$ sudo service unbound restart
```

Finally, add logging to unbound.

We want these part to be contained in the config file `/etc/unbound/unbound.conf.d/pi-hole.conf`

```txt
server:
    # If no logfile is specified, syslog is used
    logfile: "/var/log/unbound/unbound.log"
    log-time-ascii: yes
    verbosity: 1
```

We can just uncomment the logfile line if you are following the steps from above.

Then setup the dir and permissions.

```bash
sudo mkdir -p /var/log/unbound
sudo touch /var/log/unbound/unbound.log
sudo chown unbound /var/log/unbound/unbound.log
```

Update AppArmor related settings as well.

```bash
$ ls -Al  /etc/apparmor.d/local/usr.sbin.unbound
-rw-r--r-- 1 root root 0 Jul 19 02:50 /etc/apparmor.d/local/usr.sbin.unbound

# append the following line to the file
# /var/log/unbound/unbound.log rw,

$ sudo bash -c "echo '/var/log/unbound/unbound.log rw,' >> /etc/apparmor.d/local/usr.sbin.unbound"

# reload AppArmor then restart unbound
$ sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.unbound
$ sudo service unbound restart
```

I don't have AppArmor in my current pi right now so I skipped the `apparmor_parser` step.

All done now!

## Setting static IP

Make sure we are using static IP for this DNS server. Address reservation from router is still DHCP and can still cause troubles.

I suggesting setting static IP after installing Pi-hole and unbound as they need DNS to download the packages.

```bash
$ sudo nmcli connection add con-name home-wired ifname eth0 type ethernet

$ sudo nmcli con modify home-wired ipv4.method manual \
    ipv4.addresses 192.168.1.10/24 \
    ipv4.gateway 192.168.1.1

# Do we need DNS on a DNS server? I don't think so

$ sudo nmcli con down 'replace me with old connection name' && sudo nmcli con up home-wired

# If we can up the new connection without problem:
$ sudo nmcli con delete 'old connection name'
```

Don't forget to **disable DHCP for your router** and **use the DHCP from Pi-Hole instead**.

Make sure we reboot both Pi and router to see if everything works well or not.

## References

<https://github.com/pi-hole/pi-hole/#one-step-automated-install>

<https://pi-hole.net/>
