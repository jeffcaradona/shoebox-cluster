This was last updated on 2023-01-15 and test udsing a fresh Raspberry Pi OS Lite (Bullseye) install on a Pi 3B

Only do this step if using a SSD, HDD, or thumb drive. 


    $ sudo nano /etc/dphys-swapfile

    CONF_SWAPSIZE=1024


----    
    $ sudo apt update && sudo apt dist-upgrade -y
    $ sudo apt install hostapd dnsmasq
    $ sudo DEBIAN_FRONTEND=noninteractive apt install -y netfilter-persistent iptables-persistent
    $ sudo reboot
----

----
    $ sudo nano /etc/systemd/network/bridge-br0.netdev

    [NetDev]
    Name=br0
    Kind=bridge
    #EOF
    
    $ sudo nano /etc/systemd/network/br0-member-eth0.network

    [Match]
    Name=eth0
    [Network]
    Bridge=br0
    # EOF

    $ sudo systemctl enable systemd-networkd


    $ sudo nano /etc/dhcpcd.conf

    denyinterfaces wlan0 eth0

    interface br0
        static ip_address=192.168.50.1/24
        static routers=192.168.50.1
        static domain_name_servers=8.8.8.8

        interface eth1
    # EOF


    $ sudo nano /etc/sysctl.d/routed-ap.conf 

    # Enable IPv4 routing
    net.ipv4.ip_forward=1
    # EOF



    $ sudo iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE
    $ sudo iptables -A FORWARD -i eth1 -o br0 -m state --state RELATED,ESTABLISHED -j ACCEPT
    $ sudo iptables -A FORWARD -i br0 -o eth1 -j ACCEPT
    $ sudo netfilter-persistent save


    $ sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
    $ sudo nano /etc/dnsmasq.conf
    # listening interface br0 covers traffic from eth0 and wlan0
    interface=br0
    # ignore the WAN #eth1
    except-interface=eth1
    listen-address=192.168.50.1

    # logging
    log-facility=/var/log/dnsmasq.log
    log-queries

    # disables dnsmasq reading any other files like /etc/resolv.conf for nameservers
    no-resolv

    # Upstream Name Servers
    server=1.1.1.1@eth1
    server=1.0.0.1@eth1
    server=8.8.8.8@eth1
    server=8.8.4.4@eth1


    #This is the only DHCP server int he cluster
    dhcp-authoritative
    # Pool of IP addresses served via DHCP
    dhcp-range=192.168.50.100,192.168.50.199,255.255.255.0,24h

    # Add local-only domains here, queries in these domains are answered
    # from /etc/hosts or DHCP only.
    local=/sb.cluster/

    # Set the domain for dnsmasq. this is optional, but if it is set, it
    # does the following things.
    # 1) Allows DHCP hosts to have fully qualified domain names, as long
    #     as the domain part matches this setting.
    # 2) Sets the "domain" DHCP option thereby potentially setting the
    #    domain of all systems configured by DHCP
    # 3) Provides the domain part for "expand-hosts"
    domain=sb.cluster,192.168.50.0/24

    # Alias for this router
    address=/sb.cluster/127.0.0.1
    address=/sb.cluster/192.168.50.1


    # DHCP Static Assignments
    dhcp-host=node00,192.168.50.10
    dhcp-host=node01,192.168.50.11
    dhcp-host=node02,192.168.50.12
    
    # EOF

    $ sudo nano /etc/wpa_supplicant/wpa_supplicant.conf
    ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
    update_config=1
    country=US
    # EOF

    $ sudo rfkill unblock wlan

    $ sudo nano /etc/hostapd/hostapd.conf

    country_code=US
    interface=wlan0
    bridge=br0
    ssid=ShoeBox
    hw_mode=g
    channel=7
    macaddr_acl=0
    auth_algs=1
    ignore_broadcast_ssid=0
    wpa=2
    wpa_passphrase=Password_No_Quotes
    wpa_key_mgmt=WPA-PSK
    wpa_pairwise=TKIP
    rsn_pairwise=CCMP
    # EOF



    $ sudo systemctl unmask hostapd
    $ sudo systemctl enable hostapd

    $ sudo reboot
