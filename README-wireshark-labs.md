# Wireshark Labs — Packet Capture and Analysis

A series of hands-on labs learning to use Wireshark, a network packet analyzer, to capture, filter, and analyze network traffic — covering everything from basic packet capture and display filtering to decrypting HTTPS traffic and identifying hosts and users on a network.

## What I Learned (and Why It Matters)

Through these Wireshark labs, I learned how to install and configure Wireshark, capture live network traffic, and analyze packets using different display filters. I also learned how to inspect protocols such as HTTP, DHCP, TCP, and IP, follow TCP streams, and save packet captures for later analysis. In later labs, I gained experience decrypting HTTPS traffic using session keys and identifying hosts and users by analyzing IP addresses, MAC addresses, and user-agent information. These skills are important because they help cybersecurity professionals troubleshoot network problems, detect suspicious activity, investigate malware, and better understand how data moves across a network.

## Deployment Instructions

To complete these labs, first install Wireshark and launch the program. Select the correct network interface, start capturing packets, and generate network traffic by browsing websites. Use display filters such as `http`, `dhcp`, `port 80`, `ip.src`, and `ip.dst` to analyze specific traffic. Examine packet details, follow TCP streams, and save the capture as a `.pcap` file for future analysis. For advanced exercises, configure Wireshark with the required session key log file to decrypt HTTPS traffic and use filtering techniques to identify network hosts and user activity.

## What I Enjoyed Most

The part I enjoyed most was using Wireshark to capture and analyze real network traffic. It was interesting to see how different protocols communicate and how filters made it easier to focus on specific packets. I also found the HTTPS decryption and host identification labs especially engaging because they demonstrated how cybersecurity professionals investigate encrypted traffic and identify devices on a network. These hands-on activities made the concepts easier to understand and showed how Wireshark can be used in real-world cybersecurity investigations.
