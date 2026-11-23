

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

One of the most useful lessons from this experience was that troubleshooting is not always about immediately identifying the root cause. It is also about reducing uncertainty and testing assumptions.

When Azure Bastion repeatedly failed to establish an RDP session with `DC01`, I broke the problem into several parts:

* <b>Azure networking:</b> I checked whether the network configuration should allow traffic to reach `DC01`, including its private IP address, virtual network, subnet, and network security rules.
* <b>DC01:</b> I checked whether the VM and Windows were prepared to accept an RDP connection, including the Azure VM Agent, RDP configuration, Remote Desktop Services, the RDP listener, and Windows firewall rules.
* <b>Azure Bastion:</b> I checked whether Bastion itself was provisioned successfully, compared the different errors it returned, recreated the Bastion deployment, and tested whether the behavior changed.

As those checks accumulated, the major components I could verify appeared to be working correctly. `DC01` was listening for RDP, Remote Desktop Services was running, the relevant firewall rules allowed RDP traffic, the network configuration appeared appropriate, and Azure could still communicate with the guest operating system through the VM Agent. Meanwhile, Bastion was producing inconsistent errors.
### 1. Checking the Azure Networking Layer

I started with the Azure networking configuration because before troubleshooting RDP inside Windows, I wanted to establish whether there was an obvious reason that traffic should not be able to reach `DC01`.

In Azure, I confirmed that `DC01` was connected to `HelpDesk-Lab-VNet` on `Server-Subnet` and had the private IP address `10.0.1.4`. The VM intentionally did **not** have a public IP address because I wanted administrative access to go through Azure Bastion rather than expose RDP directly to the Internet.

I also reviewed the Network Security Group attached to the VM. The inbound rules included Azure's `AllowVnetInBound` rule, which allowed traffic within the virtual network. This was important because Bastion needed a network path to the VM's private address. The absence of a public IP was therefore intentional and was not, by itself, evidence of a networking problem.

<img width="1434" height="777" alt="Screenshot 2026-10-29 at 2 52 37 PM" src="https://github.com/user-attachments/assets/a87a4d16-e093-414b-b1c3-83ed9c956725" />

*Azure network settings for `DC01`, showing private IP `10.0.1.4`, no public IP address, the `HelpDesk-Lab-VNet / Server-Subnet` network assignment, and the Network Security Group inbound rules.*

This check did not prove that RDP itself was working. It only established that I could not see an obvious Azure network configuration problem that explained the timeout. That distinction helped determine the next step: instead of changing the network configuration without evidence, I needed to investigate whether `DC01` itself was actually prepared to accept an RDP connection.

The Azure networking looked reasonable, so the next question was: <b>Is the problem actually inside `DC01`?</b>

There was just one inconvenient detail: <b>I couldn't RDP into the machine I needed to troubleshoot.</b>


### 2. Checking DC01 Itself

Once I had not found an obvious problem with the Azure network configuration, I turned my attention to `DC01` itself. The next question was whether Windows was actually prepared to accept an RDP connection.

This presented an immediate problem: RDP was the connection method that was failing, so I could not simply log into the server and troubleshoot it normally. Instead, I used **Azure Run Command**, which communicates with the operating system through the Azure VM Agent and allowed me to run troubleshooting commands inside `DC01` without an RDP session.

<img width="1381" height="709" alt="Screenshot 2026-10-29 at 2 54 23 PM" src="https://github.com/user-attachments/assets/07d6f535-f8b0-4754-b66c-474b6c83e0a1" />

*Azure Run Command for `DC01`. The Azure portal explains that Run Command uses the VM Agent to execute scripts inside the virtual machine and can be used for troubleshooting and recovery. The available commands included tools for checking IP configuration, RDP settings, Remote PowerShell, and other VM configuration.*

The fact that I could use Run Command was useful evidence by itself. Azure still had a functioning management path into the guest operating system even though Bastion could not establish the RDP session. It also gave me a way to begin testing the Windows side of the connection rather than changing settings blindly.
Azure could communicate with `DC01` through the VM Agent. **But was RDP actually configured correctly inside Windows?**

Azure Run Command included an `RDPSettings` troubleshooting option specifically for checking the VM's Remote Desktop configuration. According to the description in the Azure portal, the command checks registry settings and domain policy settings and can restore relevant settings to their default values.

I ran `RDPSettings` because this allowed me to test another assumption: the network path might be available, but Windows could still be configured in a way that prevented Remote Desktop from accepting the connection.

<img width="1425" height="757" alt="Screenshot 2026-10-29 at 2 56 09 PM" src="https://github.com/user-attachments/assets/07cd2b3c-28a5-4fb6-aace-79e9a6aa6556" />


*Azure Run Command with `RDPSettings` selected. The tool checks RDP-related registry and domain policy settings and can modify the settings to their default values.*

This gave me more evidence that the RDP configuration itself was not providing an obvious explanation for the failure. But RDP configuration was only one part of the Windows side of the connection. **Even if RDP was configured correctly, was the service responsible for Remote Desktop actually running?**

The RDP configuration looked reasonable. **But was the service responsible for Remote Desktop actually running?**

The next thing I checked was **Remote Desktop Services (`TermService`)**. Windows can have the correct RDP configuration and network settings, but if the service responsible for accepting Remote Desktop connections is stopped, there is nothing listening for the incoming RDP session.

Using Run Command, I checked the service with:

`Get-Service TermService`

The result showed that `TermService` was **Running**.

This eliminated another straightforward explanation for the failed connection. The problem was not simply that Remote Desktop Services had stopped or failed to start.

But a running service still did not prove that `DC01` was actually listening for RDP connections on the network. That raised the next question: **Was Windows actually listening on TCP port 3389?**

Remote Desktop Services was running. **But was Windows actually listening for an RDP connection?**

A running service does not necessarily mean that the expected network port is open and listening. Since standard RDP uses **TCP port 3389**, I checked whether `DC01` was actually listening on that port.

The results showed RDP listening on both `0.0.0.0:3389` and `[::]:3389`.

`0.0.0.0:3389` indicated that Windows was listening for RDP connections on the available IPv4 interfaces rather than only on a specific local address. `[::]:3389` showed the corresponding IPv6 listener.

This was an important piece of evidence. I had now established that **Remote Desktop Services was running and Windows had an active RDP listener on the expected port**. The failure was becoming harder to explain as a basic RDP service or listener problem.

There was still another layer between an incoming connection and that listener, however: **Was Windows Firewall allowing the RDP traffic through?**
`DC01` was listening on TCP port 3389. **But was Windows Firewall actually allowing RDP traffic to reach it?**

Even with Remote Desktop Services running and an active listener on port 3389, Windows Firewall could still block incoming connections before they reached the service. I therefore checked the Windows Defender Firewall rules associated with Remote Desktop.

The relevant Remote Desktop inbound rules were **enabled** and configured to **allow** the traffic.

This eliminated another simple explanation. At this point, the Windows side of the connection was looking increasingly healthy: the RDP configuration appeared appropriate, `TermService` was running, Windows was listening on TCP port 3389, and the relevant firewall rules allowed incoming RDP traffic.

That still did not prove that an RDP session could successfully travel all the way from Bastion to Windows and authenticate. But the evidence was making a basic configuration problem inside `DC01` increasingly unlikely.

So the investigation shifted again. **If Azure networking looked reasonable and `DC01` appeared ready to accept an RDP connection, what exactly was happening on the Bastion side of the connection?**

### 3. Investigating Azure Bastion

Azure networking looked reasonable, and `DC01` appeared ready to accept an RDP connection. **So what was happening with Bastion?**

I returned to the Bastion side of the connection and started with its provisioning state. Azure showed the Bastion deployment as **Succeeded**, which indicated that the Bastion resource itself had been created successfully.

But when I attempted to connect to `DC01`, Bastion returned:

`There was an error in requesting a session. Please try again. HTTP request timed out`


<img width="1000" height="738" alt="Screenshot 2026-10-29 at 2 47 26 PM" src="https://github.com/user-attachments/assets/1256a1b0-79db-4d43-860d-f1c10e392d90" />


*Azure Bastion showing a successful provisioning state for the Developer SKU while simultaneously returning an HTTP timeout when attempting to establish an RDP session with `DC01`.*

This distinction was important. **“Provisioning Succeeded” did not mean “the RDP session succeeded.”** It told me that Azure had successfully provisioned the Bastion resource, while the timeout showed that something was still going wrong during the attempt to establish the actual session.

By this point, I had also tested both the correct password and a deliberately incorrect password and received the same timeout behavior. That was another useful clue. If changing the credentials did not change the response, it suggested that the failure was occurring **before the connection reached the point where those credentials could be meaningfully evaluated**.

The problem was therefore looking less like a simple bad-password or Windows RDP configuration issue and more like a failure somewhere in the Bastion session-establishment path.

And then Bastion started giving me **different errors.**

In addition to the repeated `HTTP request timed out` message, I encountered an **“Internal server error while requesting session.”** At another point, the Bastion blade itself failed to load correctly and displayed **“Failed to fetch; error code 0.”**

The changing errors made me reconsider the assumption that there had to be one incorrect setting on `DC01` that I simply had not found yet. If the same underlying Windows configuration remained in place while the Azure-side behavior changed, I needed to consider whether the platform itself was behaving consistently.

That did not prove that Bastion or Azure was the root cause. But it mattered when considered alongside everything I had already checked. Azure networking appeared appropriate, Azure could communicate with the guest operating system through the VM Agent, the RDP configuration looked reasonable, `TermService` was running, TCP port 3389 was listening, and the relevant Windows firewall rules allowed RDP traffic. Meanwhile, the errors being returned by Bastion were changing.

At this point, I was becoming less interested in finding another Windows setting to change and more interested in testing **Bastion itself**. If Bastion was the remaining questionable part of the connection path, could I change or recreate it and see whether the behavior changed?

The inconsistent errors made Bastion itself a reasonable next suspect. **So what would happen if I recreated it?**

At this point, I did not want to keep changing settings on `DC01` when the evidence suggested that its basic RDP configuration was already correct. Making additional changes there could introduce new variables and make it harder to determine what was actually affecting the connection.

Instead, I deleted and recreated the Azure Bastion deployment while leaving the known configuration on `DC01` alone. This gave me a more controlled test: **change the component I was now questioning without simultaneously changing the component that appeared to be working correctly.**

The recreated Bastion resource provisioned successfully, but the connection problem continued. I also tried connecting through different browser sessions, including Chrome Incognito and Safari, to see whether the behavior might be related to the browser or an existing portal session. The timeout behavior persisted.

That result was frustrating, but it was still useful. Recreating Bastion had not fixed the problem, and changing browser sessions had not changed the behavior. Those possibilities became less convincing explanations, while the evidence continued to suggest that `DC01` itself should have been capable of accepting the RDP connection.

At that point, I had reached an important troubleshooting decision: **Do I keep changing things simply because the connection still does not work, or do I trust the evidence I have already collected?**


The Azure network configuration appeared appropriate. Azure could communicate with `DC01` through the VM Agent. Remote Desktop was configured, `TermService` was running, TCP port 3389 was listening, and the relevant Windows firewall rules allowed RDP traffic. I had also recreated Bastion and tested different browser sessions without changing the result.

I could have continued changing settings, but without evidence pointing toward a particular configuration problem, doing so risked introducing new variables into a system that appeared to be configured correctly. Instead, I documented what I had established and stopped troubleshooting for the day.

When I returned to the lab the next day, I tried Bastion again.

**It worked.**

I never identified a specific configuration change that explained why the connection suddenly succeeded, so I cannot claim a definitive root cause. Based on the evidence I had collected, the best conclusion I could reach was the decidedly untechnical but sometimes accurate explanation: **something was up with Azure that day.**

More importantly, the experience reinforced that troubleshooting does not always end with finding one broken setting. Sometimes the useful result is narrowing the possibilities, establishing what is working, avoiding unnecessary changes, and knowing when the available evidence no longer supports changing the system.

## Building the Active Directory Environment

With reliable access to `DC01`, I could finally get back to the reason I had built the server in the first place: **practicing Active Directory administration in a functioning domain environment.**

The first step was turning `DC01` from a standalone Windows Server into a **domain controller**. I wanted the lab to give me a place to practice the kinds of tasks that appear in real help desk and Windows administration environments: creating and managing users, organizing computers and users with Organizational Units, working with permissions and Group Policy, joining computers to a domain, troubleshooting DNS and authentication, and understanding what is actually happening when a domain user signs into a computer.

To build that environment, I installed the **Active Directory Domain Services (AD DS)** role on `DC01`. Rather than treating the installation as simply a series of screens to click through, I used the process to better understand what a domain controller actually provides and how Active Directory, DNS, users, computers, and domain authentication fit together.

### Installing Active Directory Domain Services

The first step in turning `DC01` into a domain controller was installing the **Active Directory Domain Services (AD DS)** server role.

I used **Server Manager → Add Roles and Features** and selected `DC01` as the destination server. From the available server roles, I selected **Active Directory Domain Services** and allowed Windows Server to add the management tools required to administer the role.

<img width="437" height="452" alt="Screenshot 2026-10-30 at 1 56 57 PM" src="https://github.com/user-attachments/assets/d1cbdae8-f154-4b3a-9f6d-9c6c14f6e5f3" />

*Selecting the Active Directory Domain Services role for DC01. Windows Server describes AD DS as the service that stores information about network objects and uses domain controllers to provide users access to permitted resources.*

Installing AD DS adds the Active Directory components to Windows Server, but installing the role does **not** by itself make the server a domain controller. That distinction was useful to understand: at this stage, `DC01` had the software and management tools necessary to provide Active Directory Domain Services, but I had not yet created a domain or promoted the server to a domain controller.

After the AD DS role installation completed successfully, Server Manager presented the next step: **“Promote this server to a domain controller.”**

<img width="796" height="576" alt="Screenshot 2026-10-30 at 2 46 07 PM" src="https://github.com/user-attachments/assets/232226ab-f203-4207-9fac-fea31a3cd40f" />

*The Active Directory Domain Services role installed successfully on `DC01`, but Windows Server still required an additional step: promoting the server to a domain controller.*

That was the transition point between simply installing a Windows Server role and actually building the Active Directory environment. The next question was considerably bigger than “Is AD DS installed?”:

### What domain was this server going to control?

Installing the AD DS role gave `DC01` the components needed to provide Active Directory Domain Services, but it still was not a domain controller. The next step was to promote it and define the Active Directory environment it would manage.

Because this was a brand-new lab rather than a server being added to an existing Active Directory environment, I selected **Add a new forest** and created the root domain `helpdesk.test`.

This distinction helped clarify what I was actually building. A **domain controller** is the server performing the Active Directory role, while the **domain** is the logical environment containing and managing directory objects such as users, computers, groups, and organizational units. `DC01` would become the first domain controller for the new `helpdesk.test` domain and forest.

<img width="763" height="571" alt="Screenshot 2026-10-30 at 2 49 59 PM" src="https://github.com/user-attachments/assets/e3f9c6dc-14bd-4ccf-a8e9-f6b027fa0e86" />

*Creating the new `helpdesk.test` forest during the promotion of `DC01` to a domain controller.*

### What services would DC01 provide?

Now that I had defined the helpdesk.test domain, the next question was: what services did DC01 actually need to provide for that domain to function?

During the domain controller configuration, I left DNS Server and Global Catalog (GC) enabled. This was an important point in understanding how Active Directory actually works. AD DS does not operate independently of DNS. Domain-joined computers need DNS not only to translate names into IP addresses, but also to locate Active Directory services such as domain controllers.

That meant DC01 was becoming more than simply “the server with Active Directory installed.” It would be both a domain controller and DNS server for helpdesk.test. Later, when I added another server to the lab, that relationship would become especially important: before MEMBER01 could successfully find and join the domain, it would need to use DC01 for DNS.

<img width="766" height="563" alt="Screenshot 2026-10-30 at 3 02 06 PM" src="https://github.com/user-attachments/assets/4d9b172c-7c4d-4f6f-abf3-7af27d114b1e" />


*Configuring DC01 as a DNS server and Global Catalog while promoting it to the first domain controller for helpdesk.test.*

### Was the DNS warning actually a problem?

The configuration wizard then presented a warning stating that a **DNS delegation could not be created because an authoritative parent zone could not be found**. A warning during domain controller promotion certainly looks like something worth investigating, but a warning does not necessarily mean something is broken.

In this case, I was creating a completely new, isolated `helpdesk.test` forest. There was no existing parent DNS infrastructure above `helpdesk.test` where a delegation needed to be created. The warning would have been more significant if I were integrating this domain into an existing DNS hierarchy, but that was not how this lab was designed.

So rather than changing the configuration simply to make the yellow warning disappear, I determined what the warning referred to and whether it applied to the environment I was actually building. In this case, **no DNS delegation was required**, and I continued with the promotion.

<img width="763" height="562" alt="Screenshot 2026-10-30 at 3 04 58 PM" src="https://github.com/user-attachments/assets/6fed3d8f-034a-4450-a01e-75fdaf86a4f6" />


*The AD DS Configuration Wizard warning that a DNS delegation could not be created. Because `helpdesk.test` was a new standalone forest without an existing parent DNS zone, no delegation was required.*

### What was I actually about to create?

Before beginning the promotion, the wizard provided a **Review Options** page summarizing the configuration. This was a useful opportunity to verify that the individual choices I had made throughout the wizard actually added up to the environment I intended to build.

The summary showed that `DC01` would become the **first Active Directory domain controller in a new forest**, with `helpdesk.test` serving as both the new domain and forest name. The NetBIOS domain name would be `HELPDESK`, the forest and domain functional levels would be **Windows Server 2025**, and `DC01` would provide both **DNS Server** and **Global Catalog** services.

Looking at the configuration as a whole helped connect the pieces. I was no longer simply installing a Windows Server role. I was creating the identity and authentication infrastructure that the rest of the lab would depend on. Future domain users, computers, groups, and organizational units would exist within `helpdesk.test`, and other machines would need to locate and communicate with `DC01` in order to use those domain services.

<img width="757" height="562" alt="Screenshot 2026-10-30 at 3 09 35 PM" src="https://github.com/user-attachments/assets/0dc1f102-4e06-4c02-8a1e-2b76ac73941a" />


*Reviewing the final configuration before promoting `DC01`: first domain controller in the new `helpdesk.test` forest, with DNS Server and Global Catalog enabled.*

### Wait — didn't `DC01` already have a static IP?

Before promotion, the AD DS Configuration Wizard ran a prerequisites check. All of the prerequisite checks passed successfully, but Windows displayed a warning that at least one physical network adapter did not have a static IP address assigned.

That initially seemed strange because I had already configured `DC01` to keep the private IP address `10.0.1.4`. The important distinction was **where that address was being controlled**.

In Azure, I had configured `10.0.1.4` as **Static on the Azure network interface**. Inside Windows, however, the network adapter could still obtain its address automatically through DHCP. Azure would continue assigning the same reserved private address to that NIC. From the guest operating system's perspective, therefore, the adapter still appeared dynamically configured even though Azure was ensuring that `DC01` retained `10.0.1.4`.

That distinction mattered for a domain controller because other machines would eventually use `DC01` as their DNS server. I needed its address to remain predictable, but I did not need to manually duplicate the Azure network configuration inside Windows simply to make the warning disappear.

The prerequisite checks had passed, the IP address was reserved at the Azure layer, and I understood why Windows was displaying the warning. **So rather than “fixing” something that was not actually broken, I continued with the promotion.**

<img width="757" height="562" alt="Screenshot 2026-10-30 at 3 09 35 PM" src="https://github.com/user-attachments/assets/09f5fda3-bed0-49f6-bb54-435006841022" />

 
 *The AD DS prerequisite check passed successfully while also displaying the warning that Windows did not see a static IP configured on the guest network adapter.*

### Promoting `DC01` to a Domain Controller

With the prerequisite checks complete, I started the promotion of `DC01` to the first domain controller for `helpdesk.test`. The promotion process installs and configures the components needed for the new Active Directory forest and then automatically restarts the server.

Because I was administering `DC01` remotely through Azure Bastion, that restart also meant temporarily losing my connection to the server. When I attempted to reconnect, Bastion reported that the target machine was unreachable and specifically referenced RDP traffic on TCP port 3389.

In this case, the timing was important. Rather than immediately assuming that the network security configuration had failed again, I had to consider what had just changed: the server had undergone a major configuration change and restarted as part of its promotion to a domain controller. A temporarily unavailable RDP service during that process was therefore different from the earlier unexplained Bastion timeouts I had been troubleshooting.

<img width="650" height="326" alt="Screenshot 2026-10-30 at 3 14 00 PM" src="https://github.com/user-attachments/assets/9750d6a3-e9aa-416c-a0f1-1e1a62b65277" />
 *Azure Bastion temporarily reporting that `DC01` was unreachable following the domain-controller promotion and restart.*

 ### Verifying the Domain Controller Promotion

Once `DC01` finished restarting, I was able to reconnect through Azure Bastion and return to Server Manager. Rather than simply assuming that the promotion had succeeded because I could log back into the server, I wanted to verify that the expected Active Directory components were actually present.

Server Manager now showed both **AD DS** and **DNS** as installed server roles. This was an important confirmation because `DC01` was no longer simply a Windows Server VM with the AD DS role files installed; it had been configured as the domain controller and DNS server for the new `helpdesk.test` domain.

This also helped reinforce the distinction between **installing the Active Directory Domain Services role** and **promoting a server to a domain controller**. Installing the role makes the necessary AD DS components available, while the promotion process actually configures the server to provide Active Directory services for a domain.

<img width="1454" height="756" alt="Screenshot 2026-10-30 at 3 19 03 PM" src="https://github.com/user-attachments/assets/7201c8e4-92ee-48ca-bacd-570e1bba1e0b" />

*Server Manager after the promotion and restart, showing the AD DS and DNS roles on `DC01`.*

### Verifying the New Active Directory Domain

My next step was to open **Active Directory Users and Computers (ADUC)** and verify that the new domain could actually be managed through the standard Active Directory administrative tools. ADUC displayed `helpdesk.test`, confirming that the domain created during the promotion was now present and accessible.

Expanding the domain also showed the default Active Directory containers, including **Computers**, **Domain Controllers**, and **Users**. At this point, I was looking at the basic structure that I would later use to create organizational units, users, and domain-joined computers for the help desk exercises.

I then opened the **Domain Controllers** container and confirmed that `DC01` appeared there as a computer object. This provided another useful verification of the promotion: Active Directory itself recognized `DC01` as a domain controller for `helpdesk.test`, rather than simply as a Windows server with the AD DS role installed.

<img width="771" height="574" alt="Screenshot 2026-10-30 at 3 24 42 PM" src="https://github.com/user-attachments/assets/65cb7819-f2ce-4c57-afe4-d4004fe802eb" />

 *Active Directory Users and Computers showing the newly created `helpdesk.test` domain.*

<img width="753" height="525" alt="Screenshot 2026-10-30 at 3 29 37 PM" src="https://github.com/user-attachments/assets/55215628-a37a-4e8c-bad7-4ff6e1defaa8" />

*The Domain Controllers container showing `DC01` as a domain controller in `helpdesk.test`.*












