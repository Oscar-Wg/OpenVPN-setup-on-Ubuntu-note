# OpenVPN setup on Ubuntu note
**Environment**
platform: Ubuntu 16.04 LTS (on [DigitalOcean](https://www.digitalocean.com/))
requried software: openvpn, easy-ras

#### Step 1 Install package
update library, and install package:
```sh
sudo apt-get update
sudo apt-get install openvpn easy-rsa
```

#### Step 2 Create CA
create our own simple certificate authority(CA) directory:
```sh
make-cadir ~/openvpn-ca
cd ~/openvpn-ca
```

### Step 3 Configure CA
To configure our CA file
```sh
sudo vim vars
```
find out the following text:
`~/openvpn-ca/vars`
```vim
export KEY_COUNTRY="US"
export KEY_PROVINCE="CA"
export KEY_CITY="SanFrancisco"
export KEY_ORG="Fort-Funston"
export KEY_EMAIL="me@myhost.mydomain"
export KEY_OU="MyOrganizationalUnit"
```
and edit the value to your information:
```sh
export KEY_COUNTRY="MO"
export KEY_PROVINCE="MO"
export KEY_CITY="Macao"
export KEY_ORG="OscarOnGit"
export KEY_EMAIL="Oscar@example.com"
export KEY_OU="OscarVPN"
```
finally, edit the `KEY_NAME` value, here we use `server`:
```sh
export KEY_NAME="server"
```

#### step 4 Build the CA
go into openvpn-ca and source vars
```sh
cd ~/openvpn-ca
source vars
```
it will show the following output if correct
```output
Output
NOTE: If you run ./clean-all, I will be doing a rm -rf on /home/sammy/openvpn-ca/keys
```
then we clear the environment and build our CA:
```sh
./clean-all
./build-ca
```
Since we filled out our information in the vars file, so just press **ENTER** through the prompts to confirm the selections is ok:
```output
Generating a 2048 bit RSA private key
..........................................................................................+++
...............................+++
writing new private key to 'ca.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [MO]:
State or Province Name (full name) [MO]:
Locality Name (eg, city) [Macao]:
Organization Name (eg, company) [OscarOnGit]:
Organizational Unit Name (eg, section) [OscarVPN]:
Common Name (eg, your name or your server's hostname) [DigitalOcean CA]:
Name [server]:
Email Address [admin@email.com]:
```

#### Step 5 Create Server Certificate, Key, and Encryption Files
generate the OpenVPN server certificate and key pair (**using the KEY_NAME we setup before**):
```sh
./build-key-server server
```
Once again, the prompts will have default values and we just feel free to press **ENTER**. Do not enter a challenge password for this step. and in the end we will have to enter y to wo question to sign commit the certificaste:
```output
Certificate is to be certified until May  1 17:51:16 2026 GMT (3650 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
```
Next, we will generate a strong Diffie-Hellman keys to use during key exchange:
```sh
./build-dh
```
Then we can generate an HMAC signature to strengthen the server's TLS integrity verification capabilities:
```sh
openvpn --genkey --secret keys/ta.key
```

#### Step 6 Generate a Client Certificate
Now, we can generate a client certificate and key pair.
We will use `client1` as the value for our first certificate/key pair for this guide.
To produce credentials without a password, to aid in automated connections, use the build-key command like this:
```sh
cd ~/openvpn-ca
source vars
./build-key-pass client1
```

#### Step 7 Configure the OpenVPN Service
These were placed within the `~/openvpn-ca/keys` directory as they were created. We need to move our CA cert and key, our server cert and key, the HMAC signature, and the Diffie-Hellman file:
```sh
cd ~/openvpn-ca/keys
sudo cp ca.crt ca.key server.crt server.key ta.key dh2048.pem /etc/openvpn
```
Next, we need to copy and unzip a sample OpenVPN configuration file into configuration directory so that we can use it as a basis for our setup:
```sh
gunzip -c /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz | sudo tee /etc/openvpn/server.conf
```
##### Adjust the OpenVPN Configuration
Now that our files are in place, we can modify the server configuration file:
```sh
sudo vim /etc/openvpn/server.conf
```
First, remove the `;` to uncomment the `tls-auth` line. Below this, add the key-direction parameter set to `0`:
```vim
tls-auth ta.key 0 # This file is secret
key-direction 0
```
Next, find the section on cryptographic ciphers, remove the `;` to uncomment the cipher AES-128-CBC line. And add an auth line `SHA256` to select the HMAC message digest algorithm:
```vim
cipher AES-128-CBC
auth SHA256
```
Finally, find the user and group settings and remove the `;` at the beginning of to uncomment those lines:
```vim
user nobody
group nogroup
```
##### (Optional)
We strongly recommend enable the **Redirect** setting for redirecting the client IP to VPN's IP, because it was want we want, and we can change the DNS inside the config, just remove `;` to enable them:
```vim
push "redirect-gateway def1 bypass-dhcp"

push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 8.8.4.4"
```

#### Step 8 Adjust the Server Networking Configuration
First, we go to adjust the setting by modifying the `/etc/sysctl.conf` file:
```sh
sudo vim /etc/sysctl.conf
```
Inside `/etc/sysctl.conf`, look for the line that sets `net.ipv4.ip_forward`. Remove the `#` character from the beginning of the line to uncomment that setting:
```vim
net.ipv4.ip_forward=1
```
Save and close the file when you are finished.
Then we go to adjust the UFW rules configuration:
```sh
ip route | grep default
```
Remember the interface named `eth0`
```output
default via xxx.xxx.xxx.xxx dev eth0 onlink
```
Then we open th `/etc/ufw/before.rules` file to add the relevant configuration:
```sh
sudo vim /etc/ufw/before.rules
```
Insert the config like below, remember to change the interface name `eth0` if it is different:
```vim
#
# rules.before
#
# Rules that should be run before the ufw command line added rules. Custom
# rules should be added to one of these chains:
#   ufw-before-input
#   ufw-before-output
#   ufw-before-forward
#

# START OPENVPN RULES
# NAT table rules
*nat
:POSTROUTING ACCEPT [0:0] 
# Allow traffic from OpenVPN client to eth0 (change to the interface you discovered!)
-A POSTROUTING -s 10.8.0.0/8 -o eth0 -j MASQUERADE
COMMIT
# END OPENVPN RULES
 
# Don't delete these required lines, otherwise there will be errors
*filter
```
We need to tell UFW to allow forwarded packets by default as well. To do this, we will open the `/etc/default/ufw` file:
```sh
sudo vim /etc/default/ufw
```
Inside `/etc/default/ufw`, find the `DEFAULT_FORWARD_POLICY` directive. We will change the value from `DROP` to `ACCEPT`:
```vim
DEFAULT_FORWARD_POLICY="ACCEPT"
```
Finially, we can adjest the firewall itself to allow traffice to OpenVPN.
```sh
sudo ufw allow 1194/udp
sudo ufw allow OpenSSH

sudo ufw disable
sudo ufw enable
```

#### Step 9 Start and Enable the OpenVPN Service
use systemctl to start openvpn, we will add `@server` since we name the `KEY_NAME` as server on step 3.
```sh
sudo systemctl start openvpn@server
```
To check the service status we can type this:
```sh
sudo systemctl status openvpn@server
```
If everything went well, enable the service so that it starts automatically at boot:
```sh
sudo systemctl enable openvpn@server
```

#### Step 10 Create Client Configuration Infrastructure
Create a directory structure within your home directory to store the files:
```sh
mkdir -p ~/client-configs/files
chmod 700 ~/client-configs/files
```
Next, let's copy and edit an example client configuration into our directory to use as our base configuration:
```sh
cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf ~/client-configs/base.conf
vim ~/client-configs/base.conf
```
Inside `~/client-configs/base.conf`, frst locate the `remote` directive, and place the IP address of you VPN Server, and port number, port number default is `1194`
```vim
# The hostname/IP and port of the server.
# You can have multiple remote entries
# to load balance between the servers.
remote [server_IP_address] [port_num]
```
Next, uncomment the user and group directives by removing the `;`:
```vim
# Downgrade privileges after initialization (non-Windows only)
user nobody
group nogroup
```
Find the directives that set the `ca`, `cert`, and `key`. Comment out these directives since we will be adding the certs and keys within the file itself:
```vim
# SSL/TLS parms.
# See the server config file for more
# description.  It's best to use
# a separate .crt/.key file pair
# for each client.  A single ca
# file can be used for all clients.
#ca ca.crt
#cert client.crt
#key client.key
```
Mirror the `cipher`, `auth` and `key-direction` settings that we set in the `/etc/openvpn/server.conf` file, and key-direction must be set to `1`:
```vim
cipher AES-128-CBC
auth SHA256

key-direction 1
```
Save the file when you are finished.
Then we can create and open a file called make_config.sh within the ~/client-configs directory:
```sh
vim ~/client-configs/make_config.sh
```
Inside, paste the following script:
```sh
#!/bin/bash

# First argument: Client identifier

KEY_DIR=~/openvpn-ca/keys
OUTPUT_DIR=~/client-configs/files
BASE_CONFIG=~/client-configs/base.conf

cat ${BASE_CONFIG} \
    <(echo -e '<ca>') \
    ${KEY_DIR}/ca.crt \
    <(echo -e '</ca>\n<cert>') \
    ${KEY_DIR}/${1}.crt \
    <(echo -e '</cert>\n<key>') \
    ${KEY_DIR}/${1}.key \
    <(echo -e '</key>\n<tls-auth>') \
    ${KEY_DIR}/ta.key \
    <(echo -e '</tls-auth>') \
    > ${OUTPUT_DIR}/${1}.ovpn
```
Mark the file as executable by typing:
```sh
chmod 700 ~/client-configs/make_config.sh
```
#### Step 11 Generate Client Configurations
we can call the sh we create on step 10 to make our own client key pair file:
```sh
cd ~/client-configs
./make_config.sh client1
```
after that, the `client1.ovpn` will created in `~/client-configs/files`.
```sh
ls ~/client-configs/files
```
Then we can import the `ovpn` file to Windows, OSX, iOS or Android to enjoy the VPN.
