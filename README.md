# Raspberry Pi configuration

## Install software

```
sudo apt-get update
sudo apt-get install hostapd isc-dhcp-server iptables-persistent
```

## Setup DHCP server

##### /etc/dhcp/dhcpd.conf
```
# The ddns-updates-style parameter controls whether or not the server will
# attempt to do a DNS update when a lease is confirmed. We default to the
# behavior of the version 2 packages ('none', since DHCP v2 didn't
# have support for DDNS.)
ddns-update-style none;

# If this DHCP server is the official DHCP server for the local
# network, the authoritative directive should be uncommented.
authoritative;

# Use this to send dhcp log messages to a different log file (you also
# have to hack syslog.conf to complete the redirection).
log-facility local7;

subnet 192.168.42.0 netmask 255.255.255.0 {
	range 192.168.42.10 192.168.42.50;
	option broadcast-address 192.168.42.255;
	option routers 192.168.42.1;
	default-lease-time 600;
	max-lease-time 7200;
	option domain-name "local";
	option domain-name-servers 8.8.8.8, 8.8.4.4;
}
```

##### /etc/default/isc-dhcp-server

```
# On what interfaces should the DHCP server (dhcpd) serve DHCP requests?
#	Separate multiple interfaces with spaces, e.g. "eth0 eth1".
INTERFACES="wlan0"
```

## Setup interfaces

Run `sudo ifdown wlan0` to bring the interface down if it's up.

##### /etc/network/interfaces

```
# interfaces(5) file used by ifup(8) and ifdown(8)

# Please note that this file is written to be used with dhcpcd
# For static IP, consult /etc/dhcpcd.conf and 'man dhcpcd.conf'

# Include files from /etc/network/interfaces.d:
source-directory /etc/network/interfaces.d

auto lo
iface lo inet loopback

auto eth0
allow-hotplug eth0
iface eth0 inet dhcp
 
allow-hotplug wlan0

iface wlan0 inet static
  address 192.168.42.1
  netmask 255.255.255.0
```

Assign the static IP address to the wifi adapter 

```
sudo ifconfig wlan0 192.168.42.1
```

## Configure access point

##### /etc/hostapd/hostapd.conf

```
interface=wlan0
driver=rtl871xdrv
ssid=GlamCocks
country_code=US
hw_mode=g
channel=6
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=*PASSWORD*
wpa_key_mgmt=WPA-PSK
wpa_pairwise=CCMP
wpa_group_rekey=86400
ieee80211n=1
wme_enabled=1
```

##### /etc/default/hostapd

```
# Defaults for hostapd initscript
#
# See /usr/share/doc/hostapd/README.Debian for information about alternative
# methods of managing hostapd.
#
# Uncomment and set DAEMON_CONF to the absolute path of a hostapd configuration
# file and hostapd will be started during system boot. An example configuration
# file can be found at /usr/share/doc/hostapd/examples/hostapd.conf.gz
#
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

##### /etc/init.d/hostapd

```
#!/bin/sh

### BEGIN INIT INFO
# Provides:		hostapd
# Required-Start:	$remote_fs
# Required-Stop:	$remote_fs
# Should-Start:		$network
# Should-Stop:
# Default-Start:	2 3 4 5
# Default-Stop:		0 1 6
# Short-Description:	Advanced IEEE 802.11 management daemon
# Description:		Userspace IEEE 802.11 AP and IEEE 802.1X/WPA/WPA2/EAP
#			Authenticator
### END INIT INFO

PATH=/sbin:/bin:/usr/sbin:/usr/bin
DAEMON_SBIN=/usr/sbin/hostapd
DAEMON_DEFS=/etc/default/hostapd
DAEMON_CONF=/etc/hostapd/hostapd.conf
NAME=hostapd
DESC="advanced IEEE 802.11 management"
PIDFILE=/run/hostapd.pid

[ -x "$DAEMON_SBIN" ] || exit 0
[ -s "$DAEMON_DEFS" ] && . /etc/default/hostapd
[ -n "$DAEMON_CONF" ] || exit 0

DAEMON_OPTS="-B -P $PIDFILE $DAEMON_OPTS $DAEMON_CONF"

. /lib/lsb/init-functions

case "$1" in
  start)
	log_daemon_msg "Starting $DESC" "$NAME"
	start-stop-daemon --start --oknodo --quiet --exec "$DAEMON_SBIN" \
		--pidfile "$PIDFILE" -- $DAEMON_OPTS >/dev/null
	log_end_msg "$?"
	;;
  stop)
	log_daemon_msg "Stopping $DESC" "$NAME"
	start-stop-daemon --stop --oknodo --quiet --exec "$DAEMON_SBIN" \
		--pidfile "$PIDFILE"
	log_end_msg "$?"
	;;
  reload)
  	log_daemon_msg "Reloading $DESC" "$NAME"
	start-stop-daemon --stop --signal HUP --exec "$DAEMON_SBIN" \
		--pidfile "$PIDFILE"
	log_end_msg "$?"
	;;
  restart|force-reload)
  	$0 stop
	sleep 8
	$0 start
	;;
  status)
	status_of_proc "$DAEMON_SBIN" "$NAME"
	exit $?
	;;
  *)
	N=/etc/init.d/$NAME
	echo "Usage: $N {start|stop|restart|force-reload|reload|status}" >&2
	exit 1
	;;
esac

exit 0
```

## Configure network address translation

##### /etc/sysctl.conf

```
#
# /etc/sysctl.conf - Configuration file for setting system variables
# See /etc/sysctl.d/ for additional system variables.
# See sysctl.conf (5) for information.
#

#kernel.domainname = example.com

# Uncomment the following to stop low-level messages on console
#kernel.printk = 3 4 1 3

##############################################################3
# Functions previously found in netbase
#
# Uncomment the next line to enable packet forwarding for IPv4
net.ipv4.ip_forward=1

# Uncomment the next line to enable packet forwarding for IPv6
#  Enabling this option disables Stateless Address Autoconfiguration
#  based on Router Advertisements for this host
net.ipv6.conf.all.forwarding=1
```

Run `sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"` 

## Update hostapd

`wget https://github.com/GlamCocks/server-configuration/raw/master/hostapd
sudo mv /usr/sbin/hostapd /usr/sbin/hostapd.ORIG
sudo mv hostapd /usr/sbin
sudo chown root:root /usr/sbin/hostapd
sudo chmod 755 /usr/sbin/hostapd`

## Remove WPA-Supplicant

`sudo mv /usr/share/dbus-1/system-services/fi.epitest.hostap.WPASupplicant.service ~/`

## Test

Reboot and then `sudo /usr/sbin/hostapd /etc/hostapd/hostapd.conf`

## Creating service

```sudo service hostapd start
sudo service isc-dhcp-server start```

You can check the status with `sudo service hostapd status` or `sudo service isc-dhcp-server status`

```sudo update-rc.d hostapd enable
sudo update-rc.d isc-dhcp-server enable```
