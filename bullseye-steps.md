    $ sudo apt update && sudo apt dist-upgrade -y
    $ sudo apt install hostapd dnsmasq
    $ sudo DEBIAN_FRONTEND=noninteractive apt install -y netfilter-persistent iptables-persistent

    $ sudo nano /etc/systemd/network/br0-member-eth0.network

    > [Match]
    > Name=eth0
    > [Network]
    > Bridge=br0
    > # EOF

    $ sudo nano /etc/systemd/network/bridge-br0.netdev

    > [NetDev]
    > Name=br0
    > Kind=bridge
    > #EOF

    $ sudo systemctl enable systemd-networkd


    $ sudo nano /etc/dhcpcd.conf

    > denyinterfaces wlan0 eth0

    > interface br0
    >   static ip_address=192.168.50.1/24
    >   static routers=192.168.50.1
    >   static domain_name_servers=8.8.8.8

    >   interface eth1
    > # EOF


    $ sudo nano /etc/sysctl.d/routed-ap.conf 

    > # Enable IPv4 routing
    > net.ipv4.ip_forward=1
    > # EOF

    $ sudo iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE
    $ sudo iptables -A FORWARD -i eth1 -o br0 -m state --state RELATED,ESTABLISHED -j ACCEPT
    $ sudo iptables -A FORWARD -i br0 -o eth1 -j ACCEPT
    $ sudo netfilter-persistent save


    $ sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
    $ sudo nano /etc/dnsmasq.conf
    > # listening interface
    > interface=br0
    > listen-address=192.168.50.1
    > expand-hosts

    > bind-dynamic
    > domain-needed

    > # Upstream Name Servers
    > server=1.1.1.1
    > server=1.0.0.1

    > # Pool of IP addresses served via DHCP
    > dhcp-range=192.168.50.100,192.168.50.199,255.255.255.0,24h

    > # Local wireless DNS domain
    > domain=sb.cluster

    > # Alias for this router
    > address=/sb.cluster/127.0.0.1
    > address=/sb.cluster/192.168.50.1

    > # DHCP Static Assignments
    > dhcp-host=node00,192.168.50.10
    > dhcp-host=node01,192.168.50.11
    > dhcp-host=node02,192.168.50.12
    > # EOF

    $ sudo nano /etc/wpa_supplicant/wpa_supplicant.conf
    > ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
    > update_config=1
    > country=US
    > # EOF

    $ sudo rfkill unblock wlan

    $ sudo nano /etc/hostapd/hostapd.conf

    > country_code=US
    > interface=wlan0
    > bridge=br0
    > ssid=ShoeBox
    > hw_mode=g
    > channel=7
    > macaddr_acl=0
    > auth_algs=1
    > ignore_broadcast_ssid=0
    > wpa=2
    > wpa_passphrase=Password_No_Quotes
    > wpa_key_mgmt=WPA-PSK
    > wpa_pairwise=TKIP
    > rsn_pairwise=CCMP
    > # EOF



    $ sudo systemctl unmask hostapd
    $ sudo systemctl enable hostapd

    $ sudo systemctl reboot
