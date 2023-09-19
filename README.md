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

Next the playbook will pull in roles/main.yml as the next task.  The main.yml's sole purpose is to call additional tasks.  I've labeled them in order of operation by prefixing them with numbers.  This playbook is meant to have every task read and be personalized.  For example, you may not want to PermitRootLogin, so you would want to change the option in 0-gather_variables.yml

0-gather_varibles.yml - This was originally designed to gather variables as a sort of "pre-task."  
1-install packages.yml - Self-explanatory
2-configure basic routing.yml - IP forwarding options, and copy network interface config, DNS, and Firewall options to the proper location.
3-install custom services.yml - Optional services that are not required for a basic router.  Example: Setting up Dynamic DNS client.
4-ansible-pull setup.yml - this is an example of using ansible-pull to automatically update your router once you push the config to a git-repository.  Likely OK to comment this out for most users.
5-Restart services.yml - Restart any service if there were any changes made to that service.

Outside of the tasks folder, we have:
- templates, which hosts all of the files that will be copied to the appropiate linux path.  This is where each service will pull its config from.
- vars/main/xxx.yml - this is how you'll manage the router on a day to day basis.  If you need to add a port forward, you'll update port_forward.yml  DHCP/DNS/Port Forwards/Routes and Firewall source-ip filters are maintained here.
- Also be aware of groups_var/all/vars.yml and vault.yml.  The variables are set in vars.yml and vault.yml is secured with ansible-vault. (Unlock the vault with "ansible-vault decrypt group_vars/all/vault.yml" and reencrypt with "ansible-vault encrypt group_vars/all/vault.yml"
- Please be aware of the ansible.cfg file.  This is what sets the playbook default vault and the vaults password.  Notice that it calls vault_password.sh
- vault_password.sh will reference an environment variable $MYPASS -- configure this in ~/.profile at the top with an export command such as: "export MYPASS=123456789"


  # Results
  GRC Shields up should result with all ports being "Stealth"
  ![image](https://github.com/netnem/ansible-router/assets/32517635/f2f21b25-0b67-413a-8240-b37e24d237f1)

  The hardware I'm running on is a fitlet2 (Intel Celeron J3455.  Easily caps out my 1G connection:

  ![image](https://github.com/netnem/ansible-router/assets/32517635/25fd4649-29fa-4a10-9376-eeab30f76631)

  
  





