
# Azure Active Directory Help Desk Lab

## Introduction

I created this lab to get more hands-on experience with Windows Server, Active Directory, and the kinds of troubleshooting that may come up in an IT support environment. I built the environment in Microsoft Azure so I could also practice configuring virtual machines, networking, and secure remote access.

At this point, the lab consists of a Windows Server 2025 domain controller (`DC01`) running Active Directory Domain Services and DNS, and a second Windows Server (`MEMBER01`) joined to the `helpdesk.test` domain. I plan to continue expanding the environment and adding troubleshooting exercises as I practice Active Directory, Windows administration, networking, and end-user support.

Building the environment did not always go as planned. I encountered problems getting Azure Bastion working, learned more about how Active Directory interacts with domain-joined computers and local security groups, and eventually troubleshot an RDP authentication problem involving a domain user. This walkthrough documents both the configuration and the troubleshooting process, including some of the questions, assumptions, and hypotheses I worked through along the way.

## Building the Azure Lab Environment


My original goal was fairly simple: create a Windows Server virtual machine in Azure and use it to build an Active Directory lab.

I quickly discovered that creating the VM was only part of the job. Before the server had somewhere to live, I needed to set up some of the Azure infrastructure surrounding it, including resource groups, virtual networks, subnets, network interfaces, network security groups, and private versus public IP addressing.

This turned out to be useful because it made the lab more than just an Active Directory exercise. I was also gaining a deeper understanding of how the Windows systems I wanted to administer fit into the underlying network and cloud infrastructure.

I created a resource group called `HelpDesk-Lab-RG` to contain the lab resources and a virtual network called `HelpDesk-Lab-VNet`. Within that network, I created `Server-Subnet` using the `10.0.1.0/24` address range.

My first server, `DC01`, would eventually become the domain controller and DNS server for the `helpdesk.test` domain.

One of my goals was to avoid simply exposing Remote Desktop directly to the Internet. I therefore created the VM without a public IP address and planned to use Azure Bastion for remote administrative access.

At this point, my supposedly simple plan to "create a Windows server and practice Active Directory" had already become a lesson in Azure networking.

That would not be the last time the scope of this project expanded.

### Secure Remote Access with Azure Bastion

Because `DC01` was going to become a domain controller and DNS server, I wanted it to have a consistent private IP address. I configured its Azure network interface to retain the private address `10.0.1.4`. I also deliberately chose not to assign the server a public IP address.

<img width="1413" height="686" alt="Screenshot 2026-10-29 at 2 34 07 PM" src="https://github.com/user-attachments/assets/0746deb2-0e5d-4288-9f34-f4e50cfde4fb" />


This created my next problem: I had built a Windows server that I intentionally could not reach directly from the Internet, but I still needed to log into it to configure Windows Server, install Active Directory Domain Services, and eventually promote it to a domain controller.

My solution was Azure Bastion, which would allow me to establish an RDP session to the server through Azure without exposing RDP directly to the Internet.

My first attempt at configuring Bastion was more complicated — and potentially more expensive — than I expected. While working through the available options, I discovered that Azure also offered a Developer SKU intended for development and testing scenarios. That was much better suited to a small learning lab, but it was only available in supported Azure regions.

<img width="1561" height="1007" alt="ChatGPT Image Nov 22, 2026, 05_57_48 PM" src="https://github.com/user-attachments/assets/19276306-1c76-4643-95a0-6e1e9bdf1232" />


*My original East US deployment offered Premium, Standard, and Basic Bastion SKUs, but not the Developer SKU I wanted to use for this lab.*


Rather than changing the security design by giving `DC01` a public IP address, I checked where Bastion Developer was available and rebuilt the lab in a supported region, East US 2.

This was my first substantial troubleshooting detour in the project. What initially looked like a remote-access problem became a lesson in Azure regions, service availability, cost management, and the difference between designing something that simply works and designing something appropriate for the environment.

Unfortunately, moving the lab was not the end of my adventures with Azure Bastion. After rebuilding the lab in East US 2, I was able to deploy the Developer SKU successfully. I expected this to solve the remote-access problem, but attempts to establish a Bastion session with `DC01` repeatedly timed out.

<img width="1000" height="738" alt="Screenshot 2026-10-29 at 2 47 26 PM" src="https://github.com/user-attachments/assets/070f5381-1bf3-4e3a-a3bb-ac370361c19d" />

*Bastion Developer provisioned successfully, but attempts to establish an RDP session with DC01 returned an HTTP request timeout.*

### Troubleshooting the Bastion Connection

