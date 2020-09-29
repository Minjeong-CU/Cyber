---
title: "Lesson: Basic Networking"
date: "2020-08-26"
authors:
  - Michael Dresser
draft: false
tags:
  - lesson
  - linux
  - networking
---

This is a rough lesson plan/script for a meeting of the club. It covers some very basic networking concepts using `netcat` and `ssh`. This is intended to be used by a presenter, but if you missed the meeting it should be quite readable.

[Slide Deck](https://docs.google.com/presentation/d/1wnzYPGZ7xkWjJTZLPTLN6pC-22edVTayR0gjrH6Kdew/edit?usp=sharing)

## Motivation

Networking is fundamental to almost all computer systems, and therefore it is also fundamental to offensive and defensive security. A basic understanding of networking opens up the world of security methods and tools because most of them rely on networking and cannot be understood without a baseline.

### Motivating Example: A Website

Websites are the core of most people’s online experience, but how do they work? In essence, you go to a site via a domain/URL that gets translated to an IP address. An IP address is sort of like a street address - it tells you how to find a specific computer somewhere way out in the world. Your computer asks that IP address for a website, and it responds with a file that is the site. This is a basic client/server relationship. 

The server stores information and clients ask for and update this information. This concept generalizes to what are sometimes called “services” like databases, chat servers, game servers, etc.

## IP Addresses and Ports

Every computer on a network has an IP address, which is how other computers are able to locate it and communicate with it over that network. As an example, open up a terminal and type `ipconfig` (if running Windows) or `ip addr` (if running Linux). You should see a bunch of information, but in that dump you can find your computer’s IP address on the network. Find a partner. Share this address with your partner, then try running `ping 10.3.121.12` (where 10.3.121.12 is replaced with the IP of your partner’s computer). You should get a positive response, which validates that your computer and your partner’s can communicate with each other using this address.

Check if you and your partner both have “netcat” installed. If either of you don’t, get it. Now, one partner should run `netcat -l 4545`. Don’t worry, if it looks like nothing is happening, that’s fine. This establishes a “server” that is going to listen for information on port 4545. A port is just a way of differentiating information going to or from a computer - in this case, we’re asking to reserve port 4545 to be dedicated to a netcat server. Now, the other partner should run `netcat 10.3.121.12 4545` (where 10.3.121.12 is replaced with the IP of your partner’s computer). If it looks like nothing is happening, that’s good. You’ve now established a connection between your computers. Now, either partner should try typing into the terminal and pressing ENTER. The message you typed should appear in the other computer’s terminal. This is a very simple communication service that shows us a little bit of how computers communicate with each other.

## Protocols (TCP and UDP)

There is one more layer of distinction we have to address beyond IP addresses and ports -- protocols. The two main protocols by which computers communicate are TCP (Transmission Control Protocol) and UDP (User Datagram Protocol). Without going into too much detail, TCP aims to ensure all packets are received by adding some extra overhead information while UDP aims to be lightweight, simple, and fast by sacrificing the guarantee of packets being received. TCP is the dominant protocol for most applications but UDP has plenty of uses as well. Netcat uses TCP by default, but you can use UDP as well: `nc -u -l 4545` and `nc -u 10.3.121.12 4545`. Ports on computers are generally reserved by protocol as well, so you can have two different services, one running on TCP port 4545 and the other running on UDP port 4545.

## Remote Access (SSH)

Knowing how to control a machine remotely is a fundamental skill for cybersecurity hobbyists and professionals. Many defensive tools are configured from a command line or similar interface and you often won’t be able to walk over to the computer you are defending because it is in a datacenter. A huge number of offensive tools are configured and run from the command line and many exploits provide you with command line access to a remote machine.

SSH (Secure Shell) is the most common tool used to access and control computers remotely. There are other tools, like remote desktop software, but SSH is by far more common and already installed on almost every computer and server. Oftentimes exploits won’t provide true SSH access but the results will look very similar. Like the name implies, SSH allows you to get a shell (AKA terminal, AKA command line) that runs commands on a remote machine.

Let’s try it. Your computer will be the “client” and the computer you are trying to log in to will be the “server” in this interaction. Fire up a terminal like you had open before and type `ssh bandit0@bandit.labs.overthewire.org -p 2200` then hit ENTER. This means we are trying to start an SSH protocol connection (which is on top of TCP) to a machine at the name “bandit.labs.overthewire.org” (which turns into an IP address) on port 2200. We are asking to log in as user “bandit0”. You should be prompted for a password. Preface: characters you type will go through but they will not appear on the screen, this is a security feature. Type `bandit0` then hit ENTER. You should now see a line: `bandit0@bandit` and a cursor. To test that you’re on another computer, try `ip addr`. That’s all for this lesson. You have a basic understanding of networking that will serve as a foundation for most of your future security work.

## Continuation

Check out Bandit, a wargame that will help you learn basic Linux command-line skills. You’ve already logged in to level 0, let’s walk through levels 1 and 2…
https://overthewire.org/wargames/bandit/bandit0.html
