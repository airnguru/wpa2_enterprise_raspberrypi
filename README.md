# Raspberry Pi-based WPA2-Enterprise configuration

## Requirements

1. A Raspberry Pi machine with Raspbian installed (tested in a Model B
Revision 2).
1. A Wi-Fi Access Point (AP) that supports WPA2-Enterprise.

## Installation procedure

### Packages

On your Raspberry Pi machine (Pi), preferably one that has a fresh Raspbian
installation, install FreeRADIUS:

```shell
sudo apt-get update
sudo apt-get install freeradius ufw
```

### Firewall

Enable ufw and add rules to enable access to SSH and the FreeRADIUS server:

```shell
sudo ufw enable
sudo ufw allow ssh
sudo ufw allow 1812/udp
sudo ufw allow 1813/udp
```

### FreeRADIUS configuration

In the `/etc/freeradius/3.0/mods-enabled/eap` file do the following:

* Set the `default_eap_type` in the `eap` section to `ttls` (line 14).
* Set the `default_eap_type` in the `eap/ttls` section to `gtc` (search for the
section that begins with `EAP-TTLS` in the comments).

Then, create a shared secret (something random and long should suffice), and
then add it to a new RADIUS client in `/etc/freeradius/3.0/clients.conf`

```
client main {
	secret = <shared_secret>
	ipaddr = *
	require_message_authenticator = yes
}
```

Then restart the FreeRADIUS server:

```shell
sudo systemctl restart freeradius
```

### Users

FreeRADIUS allows for various user authentication methods:

* `Cleartext-Password`: This is by far the worst. Never use this.
* Hash-based schemes like `MD5-Password`, `SHA1-Password`, `SHA2-Password`,
etc., are not as secure as they claim. Even the longest hash algorithms are
vulnerable to [rainbow table](https://en.wikipedia.org/wiki/Rainbow_table)
attacks.
* The most secure alternative is `Crypt-Password`, which is based in the [scheme
6 of the UNIX crypt
function](https://administratosphere.wordpress.com/2011/06/16/generating-passwords-using-crypt3/).

For user addition, you should add users to the `/etc/freeradius/3.0/users` file.
For each user, first you generate the _crypt-ed password_ like this:

```shell
python -c 'import crypt; print(crypt.crypt("password"))'
```

Then, add the following line:

```
name.surname Crypt-Password := "<crypted-password>"
```

For example:

```
elliot.alderson := "$6$bvZC2V6LE1xPuwhS$uXFaQD1/z5EXwIQQKkvtZuhvMF75UeKdxyusOIM0wtfOW3NK9V2A6y/KxJjJLWIds2lcnBTXoREqH8FrnATKL1"
```

Save the file and then restart the FreeRADIUS server:

```shell
sudo systemctl restart freeradius
```

### AP connection setup

First thing you need to know is that the Pi machine has to have a fixed IP
address. You can do this in one of the following two ways:

1. Disabling DHCP and setting up a fixed IP address in the Pi machine.
1. Assigning an IP address from the AP via static DHCP.

One you have the Pi's IP address, go to the AP's administration module, and
it should show you the following when setting up the WPA2-Enterprise
authentication method:

* RADIUS server IP
* RADIUS server port: Use 1812 here.
* Shared Secret: The shared secret you created for the RADIUS client.

Enter this information and restart your AP.
