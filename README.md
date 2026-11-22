
# Azure Active Directory Help Desk Lab

## Introduction

I created this lab to get more hands-on experience with Windows Server, Active Directory, and the kinds of troubleshooting that may come up in an IT support environment. I built the environment in Microsoft Azure so I could also practice configuring virtual machines, networking, and secure remote access.

At this point, the lab consists of a Windows Server 2025 domain controller (`DC01`) running Active Directory Domain Services and DNS, and a second Windows Server (`MEMBER01`) joined to the `helpdesk.test` domain. I plan to continue expanding the environment and adding troubleshooting exercises as I practice Active Directory, Windows administration, networking, and end-user support.

Building the environment did not always go as planned. I encountered problems getting Azure Bastion working, learned more about how Active Directory interacts with domain-joined computers and local security groups, and eventually troubleshot an RDP authentication problem involving a domain user. This walkthrough documents both the configuration and the troubleshooting process, including some of the questions, assumptions, and hypotheses I worked through along the way.
