# Northstar Technologies — Hybrid Identity Lab

A hands-on lab building a hybrid identity pipeline across three systems commonly found side-by-side in real enterprises: on-prem Active Directory, Microsoft Entra ID, and Okta.
<img width="1983" height="793" alt="AD-Project-Image" src="https://github.com/user-attachments/assets/3000cc55-53ec-4bc9-a8fb-8855c7640467" />


## Project Goal

Build a hybrid identity environment for **Northstar Technologies** that demonstrates how identities can flow from on-premises Active Directory through Microsoft Entra ID and into Okta.

The project aims to:

- Synchronize three identities from a dedicated Active Directory OU into Microsoft Entra ID and Okta
- Preserve user attributes such as Department, Job Title, and Company throughout the identity flow
- Demonstrate identity lifecycle management, including downstream deprovisioning when a user is disabled or removed in Active Directory
- Configure Okta as the "front door" for application access, consuming identity information from Microsoft Entra ID rather than independently managing users

---

# Part 1 — Azure Infrastructure

## 1.1 Provision the Domain Controller VM

Created an Azure virtual machine to serve as the domain controller for the Northstar Technologies identity environment.

**VM Configuration:**

- **VM Name:** `vm-northstar-dc01-dev`
- **Operating System:** Windows Server 2022 Datacenter: Azure Edition
- **VM Size:** Standard D2s v4
- **Region:** East US
- **Private IP:** `172.16.0.4`
- **Public IP:** None

### Networking and Security

- Created a dedicated Azure Virtual Network:
  - **VNet:** `vnet-northstar-identity-dev`
  - **Address Space:** `172.16.0.0/16`
- Created a dedicated subnet:
  - **Subnet:** `snet-eastus-1`
  - **Address Space:** `172.16.0.0/24`
- Assigned a static private IP address (`172.16.0.4`) to the domain controller.
- No public IP was assigned to the VM to avoid exposing RDP directly to the internet.
- Azure Bastion was used to provide browser-based administrative access to the VM.
- Created a dedicated Network Security Group:
  - `nsg-northstar-identity-dev`

Using a static private IP is important for a domain controller because Active Directory and DNS rely on stable network addressing.

<img width="1436" height="762" alt="Created VM" src="https://github.com/user-attachments/assets/d428c7d3-7e81-43fe-bd27-1aff80a71f70" />

---

## 1.2 Promote the Server to a Domain Controller

Installed the following Windows Server roles and management tools:

- Active Directory Domain Services (AD DS)
- DNS Server
- Group Policy Management
- Active Directory Administrative Center
- AD DS Administrative Tools

The server was then promoted to a new Active Directory forest:

- **Forest:** `northstar.local`
- **Domain:** `northstar.local`

During promotion:

- Configured a Directory Services Restore Mode (DSRM) recovery password
- Accepted the default DNS, NetBIOS, and SYSVOL locations
- Installed and configured DNS as part of the domain controller promotion

### DNS Configuration

After promotion, the VM's DNS configuration was pointed to its own static private IP:

`172.16.0.4`

This ensures that the domain controller uses its own Active Directory-integrated DNS service rather than Azure's default DNS resolution.

---

# Part 2 — Active Directory: Users and OU Structure

## 2.1 Create the Synchronization OU

Created a dedicated Organizational Unit (OU) to scope the identities that will participate in the hybrid identity pipeline.

**OU:**

`NorthstarSyncProject`

Using a dedicated OU allows the lab to demonstrate selective synchronization rather than synchronizing the entire Active Directory environment.

---

## 2.2 Create Test Users

Created three test users inside the `NorthstarSyncProject` OU.



Created three test users (NorthstarUser1, NorthstarUser2, NorthstarUser3) with realistic organizational attributes set via the Organization tab in Active Directory - -- Users and Computers:
| User | Job Title | Department | Company |
|---|---|---|---|
| NorthstarUser1 | Northstar Engineer | Engineering | Northstar |
| NorthstarUser2 | Northstar Analyst | Analyst | Northstar |
| NorthstarUser3 | Northstar Developer | DevOPS | Northstar |

<img width="808" height="1058" alt="User1-properties" src="https://github.com/user-attachments/assets/d560b389-2198-4d21-ba90-707545a2eef9" />
<img width="806" height="1058" alt="User2-properties" src="https://github.com/user-attachments/assets/9b8ff810-59b9-4447-89d1-6dfbe17898ac" />
<img width="806" height="1064" alt="User3-properties" src="https://github.com/user-attachments/assets/bb1850ce-2e55-4980-a22b-685e32f2d4ee" />



---

## 2.3 Configure the UPN Suffix

By default, Active Directory users in the `northstar.local` forest receive a User Principal Name (UPN) ending in:

`@northstar.local`

Fix: Added an additional UPN suffix via Active Directory Domains and Trusts → Properties → UPN Suffixes:
northstar.onmicrosoft.com
Then updated each user's Account tab to use @northstar.onmicrosoft.com instead of the .local suffix — required for clean sync into Entra ID.

<img width="782" height="892" alt="UPN-suffix-fix" src="https://github.com/user-attachments/assets/a4472435-147c-4cfa-86f9-3eab058df353" />
<img width="814" height="1062" alt="UPN-suffix-fix2" src="https://github.com/user-attachments/assets/db6b87c9-fb05-4c02-af1a-fa642894a063" />

---
