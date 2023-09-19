# ansible-router
The purpose of this repository is to build a linux router for a home network and to manage it with Ansible.

# Motivation
I originally had an Ubiquiti ER-X router that I used for several years.  I loved having the debian "backend" while still having typical "Network Operating System" as the main entry into the device.  It was the perfect blend of having a Juniper-style CLI, while still having Linux backend for custom scripts.  When my ER-X started having hardware failures, I had temporarily replaced it with a two-port x86 device using pfSense.  It worked well, but it was a bit cumbersome with how I personally wanted to handle my internal DNS. I also missed having the ability to modify the config via CLI for some scripts.  While I had very few issues with pfSense, I realized that I really wanted to run Debian.  

VyOS 1.4 was a close fit, but I was ultimately annoyed by their rolling-releases. I ended up compiling v1.3 LTS and it was missing features that I needed (such as being able to write an ACLs that was bigger than a /24).  

I realized that once my home router was setup, I generally never touch it.  There was no reason for me to not set up these services manually using idempotency.  Thus this project was born.  

