# Install and Configure Apache Cloudstack Private Cloud

People behind the scenes:
- Aqshal Ilham Samudera
- Emir Fateen Haqqi
- Muhammad Nadhif Fasichul Ilmi
- Nicholas Samosir

## Introduction

## Install SSH and confirm it
```
sudo apt update
sudo apt install openssh-server -y
sudo systemctl status ssh
```
We use tailscale to remote SSH from our windows device


## Install Hardware Resource Monitoring Tools
```
apt update -y
apt upgrade -y
apt install htop lynx duf -y
apt install bridge-utils
```
This helps to see and manage what's happening inside your system in real time like CPU, RAM, Disk, Network and Processes.


## Installing Network Services and Text Editor
```
apt-get install openntpd sudo vim tar -y
apt-get install intel-microcode -y
```

Installing openntpd sets up an NTP client to synchronize the system time with servers across the internet. vim is a powerful text editor used for editing configuration files and code. tar is a tool used to compress and decompress files, and it is commonly used for extracting downloaded archives. intel-microcode is a collection of low-level instructions written in x86 assembly that improve processor functionality and security.

## Cloudstack Installation
