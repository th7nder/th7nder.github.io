## Transparent, portable VPN for digital nomads. WiFi with VPN.

You need to connect to a VPN, but you cannot install a VPN client. How can you solve this problem?

In software engineering adding just another layer of indirection is always the answer.
Instead of making a VPN connection on the device you can make it on the router and share WiFi which gives you a private internet connection.

Sounds easy enough, but what do you need to make it happen?
1. Router needs to use WiFi as its WAN interface. In other words we can connect to any WiFI network and use it as a internet source. Why? We do not always have an physical access to the access point, where we can plug in an RJ-45 cable.
2. Router must be able to talk [OpenVPN](https://openvpn.net/faq/what-is-openvpn/) or [WireGuard](https://www.wireguard.com/) protocols. Most VPN providers doesn't allow [L2TP](https://nordvpn.com/blog/l2tp-protocol/)/[PPTP](https://www.expressvpn.com/what-is-vpn/protocols/pptp), because [security reasons](https://protonvpn.com/blog/pptp/).
3. We cannot change anything on the Access Point WiFi as we don't have admin rights.


Architecture of the solution looks like this:
```
Internet <----> Access Point (AP) <--WiFi--> VPN Router (VP) <--VPN-enabled WiFi--> Any Device  
```

### Picking a router

What I did is a somewhat exotic solution and doing this on a stock firmware is not possible according to my 30 minute research.
I needed a router that can be flashed with OpenWRT and preferably under 100$.
I live in Poland and needed it ASAP, so I browsed through the websites of local electronics stores.
I sorted by price, went one by one and checked OpenWRT compatibility.
The lucky winner is: [Asus RT-AX53U](https://openwrt.org/toh/asus/rt-ax53u) which I bought [here](https://www.x-kom.pl/p/679724-router-asus-rt-ax53u-1800mb-s-a-b-g-n-ac-ax-1xusb-3xlan.html) for roughly $60.


### Stock firmware

Just to be double sure, I tried to make this work on stock firmware once I got the router.
As a base case I plugged in the device to my home router via WAN port and tried to configure VPN. So skipping the WiFi connection entirely.
This has been disqualified as it required setting up port forwarding on the [root AP](https://www.asus.com/us/support/FAQ/1033906/) and as per 3., we assume we do not have access to configure the root AP.


### Configuring VPN on a access point network (OpenWRT)

Based on:
* [Connecting to client Wi-Fi network](https://openwrt.org/docs/guide-user/network/wifi/connect_client_wifi)
* [How to use Wi-Fi as WAN connection](https://unix.stackexchange.com/questions/701346/openwrt-how-to-use-wifi-as-wan-connection)
* [How to set up Proton VPN on OpenWRT routers](https://protonvpn.com/support/how-to-set-up-protonvpn-on-openwrt-routers/)
* OpenWRT firmware version: `OpenWrt 22.03.3 r20028-43d71ad93e`

Outcome:
- Router using an existing WiFi (2.4GHz) network as internet provider (wireless WAN) 
- Exposing a new WiFi (5GHz) network with transparent VPN connection (clients do not know that it is using VPN)
- Connecting to the ProtonVPN via OpenVPN on the router


### Basic Configuration
Router configuration starts with factory defaults of OpenWRT.
My router is connected to the computer on `LAN1` port via cable. Default OpenWRT configuration does not have WiFI enabled.


1. Log into the router (`http://192.168.1.1`) with username: `root`, and an empty password.
![openwrt-login-screen](/assets/portable-vpn/1-openwrt-login.png "OpenWRT login screen")
![openwrt-welcome](/assets/portable-vpn/2-openwrt-welcome.png "OpenWRT home page")
2. Set the root password in `System -> Administration -> Router Password`

*note*: It is used to change the configuration of the router, set it wisely.
![openwrt-root-password](/assets/portable-vpn/3-router-password.png)
3. Change the default network from `192.168.1.0/24` to `192.168.166.0/24` in `Network -> Interfaces`.

*note*: Most of the default router networks are `192.168.1.0/24`, this router may receive WAN access from such network. We change it to avoid IP address conflicts. It can be changed to whatever you prefer, just make it *kind of non-default*.
* Click `Edit` button on `br-lan` interface
    ![openwrt-network-interfaces-1](/assets/portable-vpn/4-network-interfaces.png)
* Change IPv4 address to `192.168.169.1`
    ![openwrt-network-interfaces-2](/assets/portable-vpn/5-change-network.png)
* Click `Save`, `Save & Apply` and `Apply with revert after connectivity loss`.
    
    After saving, you'll need to connect to the web interface on `192.168.169.1` again and re-login to double confirm your changes.
    If you fail to do that, router will revert configuration back to `192.168.1.1`.
    ![openwrt-network-interfaces-3](/assets/portable-vpn/6-connectivity-loss.png)
    ![openwrt-network-interfaces-4](/assets/portable-vpn/7-after-connectivity.png)
    ![openwrt-network-interfaces-5](/assets/portable-vpn/8-relogin.png)

### Connecting to the Internet (via 2.4GHz interface)

0. Open `Network -> Wireless` and remove the 2.4 GHz network, it is the one under `MediaTek MT7915E 802.11axbgn`. We will use it as a WWAN.
![openwrt-network-wireless](/assets/portable-vpn/9-network-wireless.png)
![openwrt-network-wireless-2](/assets/portable-vpn/10-after-removal.png)

1. Click `Scan` on 2.4GHz interface (`radio0`) and connect to your provider network (`Join Network`).
    ![openwrt-network-wireless-3](/assets/portable-vpn/11-network-scan.png)
2. Enter network password in `WPA passphrase field`, leave everything else as default and click `Submit`.
    ![openwrt-network-wireless-4](/assets/portable-vpn/12-join-network.png)
3. You will be presented with confirmation screen. In the `Interface Configuration` add `lan` to the `Network` field. It will connect WAN and LAN, so you will also have VPN on LAN ports.
    ![openwrt-network-wireless-5](/assets/portable-vpn/13-attach-interface.png)
4. Go to `Advaned Settings` and set appropriate `Country Code` to be compliant with local regulations.
    ![openwrt-network-wireless-6](/assets/portable-vpn/14-country-code.png).
5. Click `Save`, there should be around `20 UNSAVED CHANGES`. Click `Save & Apply` and wait.
    ![openwrt-network-wireless-7](/assets/portable-vpn/15-wi-fi-changes.png).
6. Your router is now connected to the WiFi and exposes Internet via LAN!
    
### Exposing a new WiFi (via 5GHz interface)

1. Go to `Network -> Wireless`. Click `Enable` on `OpenWrt` network.
![openwrt-network-wireless-8](/assets/portable-vpn/16-enable-5ghz.png).
2. After a while, the router will expose the 5GHz network, but its not secure. Click `Edit` and change its name (I changed it to `Pizza`), then go to `Wireless Security` and set the encryption to `WPA2-PSK` and set a good password (`key`) for your new WiFi network.
![openwrt-network-wireless-9](/assets/portable-vpn/17-pizza.png).
![openwrt-network-wireless-10](/assets/portable-vpn/18-password.png).
3. Click `Save` then `Save and Apply`.
4. Your router now exposes a secure 5Ghz WiFi!

### Connecting to ProtonVPN

1. Install OpenWRT OpenVPN extensions. Go to `System -> Software` and click `Update lists`. After updating lists install 2 packages: `openvpn-openssl` and `luci-app-openvpn`.
![openwrt-network-wireless-11](/assets/portable-vpn/20-openvpn-install.png)
![openwrt-network-wireless-12](/assets/portable-vpn/21-openvpn-logs.png)
![openwrt-network-wireless-13](/assets/portable-vpn/22-openvpn-lucl.png)
2. Go to the homepage of OpenWrt web configuration. There is a new tab (`VPN`). Go to `VPN -> OpenVPN`.
![openwrt-network-wireless-14](/assets/portable-vpn/23-home.png)
![openwrt-network-wireless-15](/assets/portable-vpn/24-openvpn-conf.png)
3. Log in into your [Proton VPN account](https://account.protonvpn.com/) and download [OpenVPN configurations](https://protonvpn.com/support/vpn-config-download/) for your desired country.
4. Upload your configuration in the `OVPN configuration file upload` section by selecting a conf file, setting the instance name and clicking `Upload`.
![openwrt-network-wireless-16](/assets/portable-vpn/25-upload.png)
5. After uploading a new OpenVPN instance will show up (in my case it is `Poland`). Click `Edit`.
![openwrt-network-wireless-17](/assets/portable-vpn/26-instance.png)
6. Find line `auth-user-pass` and set it to accordingly (e.g. `/etc/openvpn/Poland.auth`). In the field below specify your ProtonVPN username and password (ProtonVPN -> Account -> OpenVPN / IKEv2 Username).
![openwrt-network-wireless-18](/assets/portable-vpn/27-password.png)
7. Click `Save` and go back to `VPN -> OpenVPN`. Click `Enabled` to yes, then `Save & Apply` and `start`.
![openwrt-network-wireless-19](/assets/portable-vpn/28-vpn-connect.png)
8. At this point, the VPN is set up and your router can use it. However, the devices in the LAN of your router won’t be able to access the Internet anymore. To do this, you need to set the VPN network interface as public by assigning a VPN interface to WAN zone.
9. Go to `Network -> Firewall` and select `Edit` on `wan => reject` section. 
![openwrt-network-wireless-19](/assets/portable-vpn/29-firewall-1.png)
* Select `Advanced Settings`, then in the `Covered devices` field, select `tun0`.
![openwrt-network-wireless-19](/assets/portable-vpn/29-firewall-2.png)
10. `Save` -> `Save & Apply` and wait.
11. Voila! You now have a WiFi and LAN ports secured by VPN. You can test it on a site like [ipleak.net](https://ipleak.net).
![openwrt-network-wireless-19](/assets/portable-vpn/30-ip.png)


### FAQ

1. How do I start over the configuration?
    * Factory reset OpenWRT by going into `System -> Backup / Flash Firmware -> Perform reset.
    * OR Turn off the router; turn it on; press Reset button for 10 seconds and wait for a few minutes.
2. How can I change my Access Point network?
    * Repeat the steps from [connecting to the internet](#connecting-to-the-internet-via-24ghz-interface) section without step 0.