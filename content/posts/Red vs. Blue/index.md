+++
title = 'Red vs. Blue'
date = 2023-12-09T19:05:03-04:00
draft = false
+++

## Systems vs. Security

During a cross-program project between systems administration and cyber security students, the cyber security students were tasked with breaking into a stand-alone server administered by the systems students. We broke off into teams and were given the IP address of our target server.

Our team chose the name 'Ping Pirates' and began enumerating the target machine. Using common tools such as nmap to find out which ports were open and what services were running, we discovered a limited range of ports and a dashboard running on a random ephemeral port. 

Among the open ports was port 21, potentially offering ftp access. We enumerated this port more closely and found that it offered anonymous login, allowing us to access a staff list which could then be used to generate a list of usernames. Once we generated a list of potential usernames following common patterns (first.last, last.first, firstinitial.last, etc.), we did some quick brute force attacks to see if there were any accounts with weak passwords. After running these brute attacks with larger wordlists, it became apparent that this would not be a viable means of gaining initial access; this was made painfully obvious when we later discovered that all passwords were above 14 characters in length, with the most privileged passwords randomly generated and up to 20 characters.

We then changed focus and tried to assess how the systems students would be using the machine and looked into intercepting traffic to gain more intel. We knew they had the standard RDP port open and that they had a dashboard for a monitoring platform. We decided the best thing to do would be to try and relay traffic to either (a) intercept a login to their monitoring platform, as it did not use https, or (b) use a tool like Responder to relay traffic and potentially gain hashes.