# ansible-router
The purpose of this repository is to build a linux router for a home network and to manage it with Ansible.

# Motivation
I originally had an Ubiquiti ER-X router that I used for several years.  I loved having the debian "backend" while still having typical "Network Operating System" as the main entry into the device.  It was the perfect blend of having a Juniper-style CLI, while still having a Linux backend for custom scripts and utilities.  When my ER-X started having hardware failures, I had temporarily replaced it with a two-port x86 device using pfSense.  It worked well, but it was a bit cumbersome with how I personally wanted to handle my internal DNS. I also missed having the ability to modify the config via CLI.  While I had very few issues with pfSense, I realized that I really missed having the Debian backend.  

VyOS 1.4 was a close fit, but I was ultimately annoyed by their rolling-releases. I ended up compiling v1.3 LTS and it was missing features that I needed (such as being able to write an ACLs that were bigger than a /24).  

I realized that my home router isn't really that complicated and once my home router was setup, I generally never touch it.  There was no reason for me to not set up these services manually using Ansible idempotent configuration. I get full control over which features/utilities that get installed, and updating the router is as simple as running apt update.  

I also liked the idea of not being bound to any specific architecture (x86 vs ARM).  While I haven't tested on ARM, as long as the packages are available, this should work.

Thus this project was born.  

# Requirements
- A device that has two ethernet ports. 
- Ansible-vault knowledge for maintaining your passwords.
- Jinja2 knowledge for customization of the config files that will get pushed to the device.

# Layout

The top-level playbook "build_linux_router.yml" will perform the following:
  - Create a dynamic inventory so that it can be ran against any IP address as an extra_var
  - Set up the connection for the role to work correctly, including pulling in the path for /sbin
  - Call the role "build-linux-router"

Next, the playbook will pull in roles/main.yml as the next task.  The main.yml's sole purpose is to call additional tasks.  I've labeled them in order of operation by prefixing them with numbers.  This playbook is meant to have every task read and be personalized.  For example, you may not want to PermitRootLogin, so you would want to change the option in 0-gather_variables.yml

- 0-gather_varibles.yml - This was originally designed to gather variables as a sort of "pre-task."  
- 1-install packages.yml - Self-explanatory
- 2-configure basic routing.yml - IP forwarding options, and copy network interface config, DNS, and Firewall options to the proper location.
- 3-install custom services.yml - Optional services that are not required for a basic router.  Example: Setting up Dynamic DNS client.
- 4-Restart services.yml - Restart any service if there were any changes made to that service.

Outside of the tasks folder, we have:
- templates, which hosts all of the files that will be copied to the appropiate linux path.  This is where each service will pull its config from.
- vars/main/xxx.yml - this is how you'll manage the router on a day to day basis.  If you need to add a port forward, you'll update port_forward.yml  DHCP/DNS/Port Forwards/Routes and Firewall source-ip filters are maintained here.
- Also be aware of groups_var/all/vars.yml and vault.yml.  The variables are set in vars.yml and vault.yml is secured with ansible-vault. (Unlock the vault with "ansible-vault decrypt group_vars/all/vault.yml" and reencrypt with "ansible-vault encrypt group_vars/all/vault.yml"
- Please be aware of the ansible.cfg file.  This is what sets the playbook default vault and the vaults password.  Notice that it calls vault_password.sh
- vault_password.sh will reference an environment variable $MYPASS -- configure this in ~/.profile at the top with an export command such as: "export MYPASS=123456789" and remove the export statement from vault_password.sh
- Alternatively, decrypt the current vault with the default password in vault_password.sh, and then change it to something unique.  Then reencrypt the vault.  You really should use environmental variables here for security reasons.


# Router Results
  GRC Shields up should result with all ports being "Stealth"
  ![image](https://github.com/netnem/ansible-router/assets/32517635/f2f21b25-0b67-413a-8240-b37e24d237f1)

  The hardware I'm running on is a fitlet2 (Intel Celeron J3455.  Easily caps out my 1G connection:

  ![image](https://github.com/netnem/ansible-router/assets/32517635/25fd4649-29fa-4a10-9376-eeab30f76631)


# Playbook Results 
  Please note, the vault username and password must be updated to correct credentials:
  Also note: the final task "restart networking service" will timeout the first time it is ran .  This is because the playbook will change the lan_ip of the target_host device. You would need to rerun the playbook with the new lan_ip for managing it.
  Tested against debian 12 virtual machine:

```
dan@dans:~/ansible-router$ ansible-playbook build_linux_router.yml -e target_host=192.168.3.102
[WARNING]: No inventory was parsed, only implicit localhost is available
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [localhost] *******************************************************************************************************************************************************************************************************************************************************************************

TASK [add hosts from extra-vars to "temp" group] ***********************************************************************************************************************************************************************************************************************************************
changed: [localhost]

PLAY [temp] ************************************************************************************************************************************************************************************************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************************************************************************************************************************************************************************
ok: [192.168.3.102]

TASK [command] *********************************************************************************************************************************************************************************************************************************************************************************
ok: [192.168.3.102]

PLAY [Include Role for temp group] *************************************************************************************************************************************************************************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************************************************************************************************************************************************************************
ok: [192.168.3.102]

TASK [include_role : build_linux_router] *******************************************************************************************************************************************************************************************************************************************************

TASK [build_linux_router : debug] **************************************************************************************************************************************************************************************************************************************************************
included: /home/dan/ansible-router/roles/build_linux_router/tasks/0-gather_variables.yml for 192.168.3.102

TASK [build_linux_router : Get IP of my_dynamic_ddns_host] *************************************************************************************************************************************************************************************************************************************
ok: [192.168.3.102]

TASK [build_linux_router : print iptables.j2] **************************************************************************************************************************************************************************************************************************************************
ok: [192.168.3.102] => 
  msg: |2-
  
    *nat
    :PREROUTING ACCEPT [0:0]
    :INPUT ACCEPT [0:0]
    :OUTPUT ACCEPT [0:0]
    :POSTROUTING ACCEPT [0:0]
  
    # enp6s19 is WAN interface, enp6s18 is LAN interface
    -A POSTROUTING -o enp6s19 -j MASQUERADE
  
    # NAT pinhole: HTTP from WAN to LAN
  
    -A PREROUTING -p tcp -m tcp -i enp6s19 --source 4.4.3.0/24 --dport 443 -m conntrack --ctstate NEW -j DNAT --to-destination 192.168.1.10:443
    -A PREROUTING -p tcp -m tcp -i enp6s19 --source 4.4.3.0/24 --dport 22 -m conntrack --ctstate NEW -j DNAT --to-destination 192.168.1.10:22
  
  
    COMMIT
  
    *filter
    :INPUT ACCEPT [0:0]
    :FORWARD ACCEPT [0:0]
    :OUTPUT ACCEPT [0:0]
  
    # Service rules
  
    # basic global accept rules - ICMP, loopback, traceroute, established all accepted
    -A INPUT -s 127.0.0.0/8 -d 127.0.0.0/8 -i lo -j ACCEPT
  
    #allow ping from inside to outside
    -A OUTPUT -p icmp --icmp-type echo-request -j ACCEPT
    -A INPUT -p icmp --icmp-type echo-reply -j ACCEPT
  
    # accept all ping from LAN
    -A INPUT -i enp6s18 -p icmp -j ACCEPT
  
    -A INPUT -m state --state ESTABLISHED -j ACCEPT
  
    # DNS - accept from LAN
    -A INPUT -i enp6s18 -p tcp --dport 53 -j ACCEPT
    -A INPUT -i enp6s18 -p udp --dport 53 -j ACCEPT
  
    # SSH - accept from LAN
    -A INPUT -i enp6s18 -p tcp --dport 22 -j ACCEPT
  
    # SNMP - accept from LAN
    -A INPUT -i enp6s18 -p udp --dport 161 -j ACCEPT
  
    # DHCP client requests - accept from LAN
    -A INPUT -i enp6s18 -p udp --dport 67:68 -j ACCEPT
  
    # PING - Accept from LAN
    -A INPUT -i enp6s18 -p icmp -j ACCEPT
  
    # drop all other inbound traffic
    -A INPUT -j DROP
  
    # Forwarding rules
  
    # forward packets along established/related connections
    -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
  
    # forward from LAN (enp6s18) to WAN (enp6s19)
    -A FORWARD -i enp6s18 -o enp6s19 -j ACCEPT
  
    # forward all LAN to LAN
    -A FORWARD -i enp6s18 -o enp6s18 -j ACCEPT
  
    # allow traffic from our NAT pinhole
  
    #allow specific source_ips to reach the port forwarded ips and ports (happens after NAT)
    -A FORWARD -p tcp --source 4.4.3.0/24 -d 192.168.1.10 --dport 443 -j ACCEPT
    -A FORWARD -p tcp --source 4.4.3.0/24 -d 192.168.1.10 --dport 22 -j ACCEPT
  
  
    # drop all other forwarded traffic
    -A FORWARD -j DROP
  
    COMMIT

TASK [build_linux_router : print interfaces.j2] ************************************************************************************************************************************************************************************************************************************************
ok: [192.168.3.102] => 
  msg: |2-
  
    auto lo
    iface lo inet loopback
  
    auto enp6s18
    iface enp6s18 inet static
            address 192.168.1.1/24
            up ip route add 10.11.12.0/24 via 192.168.2.129
            up ip route add 100.64.0.0/10 via 192.168.2.135
  
  
  
    auto enp6s19
    iface enp6s19 inet dhcp
            dns-nameservers 1.1.1.1,8.8.8.8

TASK [build_linux_router : print dnsmasq.j2] ***************************************************************************************************************************************************************************************************************************************************
ok: [192.168.3.102] => 
  msg: |-
    server=1.1.1.1
    server=8.8.8.8
  
    # Add domains which you want to force to an IP address here.
    address=/wordpress.mydomain.com/192.168.1.107
  
    #set dhcp range and router gateway
    dhcp-range=192.168.1.150,192.168.1.254,255.255.255.0,12h
    dhcp-option=3,192.168.1.1
  
    dhcp-host=aa:bb:cc:ff:55:55,my_laptop,192.168.1.50,infinite
    dhcp-host=aa:bb:cc:dd:22:23,my_other_laptop,192.168.1.51,infinite
    dhcp-host=aa:bb:cc:dd:11:22,my_phone,192.168.1.52,infinite

TASK [build_linux_router : print ddclient.j2] **************************************************************************************************************************************************************************************************************************************************
ok: [192.168.3.102] => 
  msg: |-
    use=if, if=enp6s19
    protocol=cloudflare, \
    zone=mydomain.com, \
    ttl=1,
    ssl=yes, \
    login=token \
    password='123456789' \
    mydomain.com

TASK [build_linux_router : allow root login for debian machines] *******************************************************************************************************************************************************************************************************************************
ok: [192.168.3.102]

TASK [build_linux_router : Remove deb cdrom from sources.list] *********************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : Install packages] ***************************************************************************************************************************************************************************************************************************************************
included: /home/dan/ansible-router/roles/build_linux_router/tasks/1-install packages.yml for 192.168.3.102

TASK [build_linux_router : update apt] *********************************************************************************************************************************************************************************************************************************************************
ok: [192.168.3.102]

TASK [build_linux_router : Install sudo] *******************************************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : add user to sudoers] ************************************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : Install vlan support] ***********************************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : Install dnsmasq] ****************************************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : Install ddclient] ***************************************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : Install iptables] ***************************************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : Install iptables iptables-persistent] *******************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : Install python3-pexpect] ********************************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : Install curl] *******************************************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : Install nmap] *******************************************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : Install dnstools] ***************************************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : Install snmp] *******************************************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : Install snmpd] ******************************************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : Add community to snmpd] *********************************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : Remove v4 SNMP string from snmpd.conf] ******************************************************************************************************************************************************************************************************************************
ok: [192.168.3.102]

TASK [build_linux_router : Replace agentaddress with publically accessible] ********************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : remove v6 SNMP string from snmpd.conf] ******************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : Install tcpdump] ****************************************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : install net-tools] **************************************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : Configure basic routing] ********************************************************************************************************************************************************************************************************************************************
included: /home/dan/ansible-router/roles/build_linux_router/tasks/2-configure basic routing.yml for 192.168.3.102

TASK [build_linux_router : Set net.ipv4.ip_forward to 1] ***************************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : Set net.ipv6.conf.all.forwarding to 1] ******************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : Replace /etc/network/interfaces with a template] ********************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : Replace /etc/dnsmasq with a template] *******************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : Replace /etc/iptables/rules.v4 with a template] *********************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : install custom services.yml] ****************************************************************************************************************************************************************************************************************************************
included: /home/dan/ansible-router/roles/build_linux_router/tasks/3-install custom services.yml for 192.168.3.102

TASK [build_linux_router : Configure ddclient (Dynamic DNS)] ***********************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : restart services.yml] ***********************************************************************************************************************************************************************************************************************************************
included: /home/dan/ansible-router/roles/build_linux_router/tasks/4-restart services.yml for 192.168.3.102

TASK [build_linux_router : restart /etc/sysctl.conf] *******************************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : Start ddclient service] *********************************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : restart dnsmasq service] ********************************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : apply new IP tables rules] ******************************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : Restart snmpd] ******************************************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

TASK [build_linux_router : restart sshd for debian machines] ***********************************************************************************************************************************************************************************************************************************
skipping: [192.168.3.102]

TASK [build_linux_router : restart networking service] *****************************************************************************************************************************************************************************************************************************************
changed: [192.168.3.102]

PLAY RECAP *************************************************************************************************************************************************************************************************************************************************************************************
192.168.3.102              : ok=47   changed=31   unreachable=0    failed=0    skipped=1    rescued=0    ignored=0   
localhost                  : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```





