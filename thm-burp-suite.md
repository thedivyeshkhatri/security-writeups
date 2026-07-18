# Burp Suite — [TryHackMe](https://tryhackme.com/)

**Date:** 2026-07-12
**Category:** Web
**Skills used:** Burp Suite, web request interception

## Overview
Burp Suite is a Java-based framework for web and mobile application
penetration testing. It works by sitting between your browser and the
target as a proxy, letting you intercept, inspect, and modify traffic.

## Basics
Covered the dashboard, navigation, and options, then set up Burp Proxy
with FoxyProxy to route browser traffic through Burp. Also learned
scoping and targeting — restricting Burp to only the hosts I'm
authorized to test, which keeps testing focused and in-bounds.

## Repeater
Used to modify and resend individual requests to a target, manipulate
requests captured by the proxy, or craft requests from scratch.
*When I'd use it:* manually testing how a single parameter responds to
different payloads without resending the whole flow.

## Intruder
Automates request modification for repetitive testing with varied input
values, sending many requests with altered values based on my
configuration.
*When I'd use it:* fuzzing a parameter to discover hidden values, or
testing a login form against a list of passwords.

## Other Tools
Explored Encoding/Decoding, Hashing, and the Comparer, Sequencer, and
Organizer tools.

## Extensions
Explored how Burp's functionality can be expanded with extensions.

## What I Learned
This was a great tool to explore hands-on. I learned to intercept and
modify requests, and in the labs I got practice cracking credentials —
including using a macro to crack credentials on a form protected by CSRF
tokens. I hit a roadblock getting the macro configured correctly, but
after some research I worked out the issue and got it working. Understanding
how to handle CSRF tokens during automated testing was the most useful 
thing I took from this module.
