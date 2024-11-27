# Task 1: Firewall configuration 
**Question 1**: 
Setup a set of vms/containers in a network configuration of 2 subnets (1,2) with a router forwarding traffic between them. Relevant services are also required:
- The router is initially can not route traffic between subnets
- PC0 on subnet 1 serves as a web server on subnet 1
- PC1,PC2 on subnet 2 acts as client workstations on subnet 2

**Answer 1**:
1. Network Configuration:

- Two subnets: Subnet 1 (`192.168.1.0/24`) and Subnet 2 (`192.168.2.0/24`).
- A router to connect and route traffic between the two subnets.
2. VM/Container Setup:

- Router: A Linux VM/container with two network interfaces: one for each subnet.
- Interface 1: `192.168.1.1` (connected to Subnet 1).
- Interface 2: `192.168.2.1` (connected to Subnet 2).
- PC0: A VM/container in Subnet 1 (`192.168.1.10`) configured as a web server.
- PC1 and PC2: VMs/containers in Subnet 2 (`192.168.2.10`, `192.168.2.11`), acting as client workstations.
3. Router Configuration:
  
- Initially disable IP forwarding on the router:
` echo 0 > /proc/sys/net/ipv4/ip_forward `
4. Web Server Configuration:

- Install and configure a web server (e.g., Apache or Nginx) on PC0:
` sudo apt update && sudo apt install apache2 -y
  echo "Welcome to the web server!" | sudo tee /var/www/html/index.html
  sudo systemctl start apache2 `

**Question 2**:
- Enable packet forwarding on the router.
- Deface the webserver's home page with ssh connection on PC1

**Answer 2**:
1. Enable Packet Forwarding:

- On the router, enable IP forwarding to allow traffic between subnets:
`echo 1 > /proc/sys/net/ipv4/ip_forward`
- Add IP forwarding permanently by editing `/etc/sysctl.conf`:
`net.ipv4.ip_forward = 1`
- Then reload sysctl settings:
`sudo sysctl -p`
2. Deface the Web Server:
  
- From PC1, connect to the web server on PC0 using SSH:
`ssh user@192.168.1.10`
- Edit the web server’s homepage:
`echo "Hacked by PC1!" | sudo tee /var/www/html/index.html`
- Verify the defacement by accessing `http://192.168.1.10` from PC2 or any other host.

**Question 3**:
  Config the router to block ssh to web server from PC1, leaving ssh/web access normally for all other hosts from subnet 1.   

**Answer 3**:
1. Router Firewall Rules:
- Use `iptables` on the router to block SSH traffic from PC1:
`sudo iptables -A FORWARD -p tcp --dport 22 -s 192.168.2.10 -d 192.168.1.10 -j DROP`
- Allow all other traffic:
`sudo iptables -A FORWARD -d 192.168.1.10 -p tcp --dport 22 -j ACCEPT
 sudo iptables -A FORWARD -d 192.168.1.10 -p tcp --dport 80 -j ACCEPT`
- Verify the rules:
`sudo iptables -L -v`

**Question 4**:
- PC1 now servers as a UDP server, make sure that it can reply UDP ping from other hosts on both subnets.
- Config personal firewall on PC1 to block UDP accesses from PC2 while leaving UDP access from the server intact.

**Answer 4**:
1. Configure PC1 as a UDP Server:
- Run a simple UDP server on PC1:
`sudo apt install netcat -y
 nc -u -l 12345`
- Ensure it can respond to UDP ping:
`sudo apt install iputils-arping
 arping -I eth0 192.168.2.10`

2. Router Configuration for UDP Ping:
- Ensure the router forwards UDP packets between subnets.

3. Personal Firewall on PC1:

- Block UDP access from PC2:
`sudo iptables -A INPUT -p udp --dport 12345 -s 192.168.2.11 -j DROP`
- Allow UDP access from Subnet 1:
`sudo iptables -A INPUT -p udp --dport 12345 -s 192.168.1.0/24 -j ACCEPT`
- Verify rules:
`sudo iptables -L -v`

# Task 2: Encrypting large message 
Use PC0 and PC2 for this lab 
Create a text file at least 56 bytes on PC2 this file will be sent encrypted to PC0
**Question 1**:
Encrypt the file with aes-cipher in CTR and OFB modes. How do you evaluate both cipher in terms of error propagation and adjacent plaintext blocks are concerned. 
- Demonstrate your ability to send file to PC0 to with message authentication measure.
- Verify the received file for each cipher modes

**Answer 1**:

1. Encrypt the file using AES in CTR and OFB modes:
- Create a file with at least 56 bytes on PC2:
`echo "This is a sample file with more than fifty-six bytes of content for testing." > file.txt`
- Encrypt the file in CTR mode:
`openssl enc -aes-256-ctr -in file.txt -out file_ctr.enc -K <256-bit-key> -iv <128-bit-IV>`
- Encrypt the file in OFB mode:
`openssl enc -aes-256-ofb -in file.txt -out file_ofb.enc -K <256-bit-key> -iv <128-bit-IV>`

2. Send the file to PC0 with message authentication:
- Generate a Message Authentication Code (MAC) using HMAC:
`openssl dgst -sha256 -hmac <shared-key> -out file_ctr.mac file_ctr.enc
 openssl dgst -sha256 -hmac <shared-key> -out file_ofb.mac file_ofb.enc`
- Send the encrypted files and MAC to PC0:
`scp file_ctr.enc file_ctr.mac file_ofb.enc file_ofb.mac user@PC0:/destination/directory`

3. Verify received files on PC0:
- Check integrity using HMAC:
`openssl dgst -sha256 -hmac <shared-key> -verify file_ctr.mac file_ctr.enc
 openssl dgst -sha256 -hmac <shared-key> -verify file_ofb.mac file_ofb.enc`

**Question 2**:
- Assume the 6th bit in the ciphered file is corrupted.
- Verify the received files for each cipher mode on PC0

**Answer 2**:
1. Simulate corruption:
- Corrupt the 6th bit of the ciphered file:
`xxd file_ctr.enc > file_ctr.hex
 sed -i 's/^.\{6\}/<altered-byte>/g' file_ctr.hex
 xxd -r file_ctr.hex > file_ctr_corrupted.enc`
2. Verify corrupted files on PC0:
- Verify integrity of the corrupted files using HMAC:
`openssl dgst -sha256 -hmac <shared-key> -verify file_ctr.mac file_ctr_corrupted.enc`
- HMAC verification will fail, indicating tampering in the encrypted file.

**Question 3**:
- Decrypt corrupted files on PC0.
- Comment on both ciphers in terms of error propagation and adjacent plaintext blocks criteria. 

**Answer 3**:
1. Decrypt corrupted files on PC0:
- Decrypt the corrupted file in CTR mode:
`openssl enc -d -aes-256-ctr -in file_ctr_corrupted.enc -out file_ctr_corrupted.txt -K <256-bit-key> -iv <128-bit-IV>`
- Decrypt the corrupted file in OFB mode:
`openssl enc -d -aes-256-ofb -in file_ofb_corrupted.enc -out file_ofb_corrupted.txt -K <256-bit-key> -iv <128-bit-IV>`

2. Comment on error propagation and adjacent plaintext blocks:
- CTR Mode:
  - A single bit error in the ciphertext affects only the corresponding bit in the plaintext.
  - Adjacent blocks remain unaffected, ensuring better isolation of errors.
- OFB Mode:
  - Similar to CTR mode, errors are limited to the corresponding bit in the plaintext.
  - Adjacent plaintext blocks are not affected as OFB processes blocks independently.     
