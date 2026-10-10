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

### Part 3 — Microsoft Entra Connect (AD → Entra ID Sync)
### 3.1 Download and Install
- Downloaded **Microsoft Entra Connect Sync** inside the VM signing in with my Global Administrator account.
- Entra admin center → Identity → Hybrid management → Microsoft Entra Connect → Get Started → Manage → Download Connect Sync Agent
- This requires signing in with a Global Administrator account.
### 3.2 Installation Wizard — Key Decisions
- Install type: chose Customize (not Express) — needed to scope sync to specific OUs.
- Required Components: left all optional boxes unchecked — lab doesn't need custom SQL, a custom service account, etc.
- User Sign-In: chose Password Hash Synchronization — simplest sign-in method, no ADFS needed.
- When you sign in to Entra, it will automatically sign in and will recognize the hash, but not the password since Entra does not store the actually
  password. So if you want to change the actually password it would have to be changed in AD and it will automatically sync the new hash in Entra. 
- Microsoft Entra sign-in (UPN matching): checked "Continue without matching all UPN suffixes to verified domains" — no custom domain owned/verified for this lab; users still sync, just can't sign in with that exact UPN.
- Uniquely identifying users: chose "Users are represented only once across all directories" — single-forest lab, so no cross-forest identity matching is needed.
- Domain/OU Filtering: chose Sync selected domains and OUs → checked only NorthstarSyncProject — avoids syncing built-in/default AD accounts.
- Filter users and devices: chose Synchronize all users and devices — the pilot-group filter here is a second, independent scoping mechanism; the OU filtering above already handles scope.


<img width="1760" height="1224" alt="Password-Hash-Sync" src="https://github.com/user-attachments/assets/b33d9869-c77b-4c8a-af0b-37be9196147d" />
<img width="1744" height="1226" alt="Connect-directories" src="https://github.com/user-attachments/assets/0dee58ee-739d-4ed2-b614-1acc99e6d915" />
<img width="1742" height="1224" alt="Domain-filtering" src="https://github.com/user-attachments/assets/9723fe92-efe8-4488-8b80-0e94192d9964" />
<img width="1742" height="1218" alt="Config-complete" src="https://github.com/user-attachments/assets/db0f8c09-1fc9-438e-ba2a-9805ef1350bb" />

### 3.3 Verification
- Confirmed all three users appear in Entra admin center → Users, each showing:
- On-premises sync: Yes
- Company name: Okta (and other attributes) correctly populated from AD

<img width="2860" height="1500" alt="Synced-users" src="https://github.com/user-attachments/assets/3d8db3ce-b2c5-4ea1-b9d2-6032b0269c9f" />

### Part 4 — Entra ID: Dynamic Group for Scoped Provisioning
- Rather than provisioning the entire directory (this tenant is a Microsoft 365 Developer tenant pre-seeded with ~107 demo users), created a dynamic security group to scope exactly which users get pushed downstream to Okta.
- Group: Okta Synced Users Membership type: Dynamic User Rule:
```powershell
(user.companyName -eq "Northstar") and (user.displayName -startsWith "Northstar")
```
- Confirmed all three test users landed in the group automatically. This group is later used to scope the Okta provisioning connector's assignment.

<img width="2840" height="1516" alt="Group-members" src="https://github.com/user-attachments/assets/5f5ad938-58c2-4109-854a-0438c02895eb" />

- In a production environment, an attribute-based dynamic security group would be a best practice for automatically maintaining the set of users provisioned to downstream applications such as Okta. This allows membership to update automatically as users' attributes change.
- For this lab, I used an assigned security group instead of a dynamic group. Dynamic group membership requires Microsoft Entra ID P1/P2 licensing, which would add unnecessary cost for a three-user demonstration environment.


### Part 5 — Okta: Identity Federation (Authentication Trust)
- This step establishes an authentication trust relationship between Okta and Entra ID. It allows Okta to trust Entra ID as an identity source for sign-in.
### 5.1 Create the Entra App Registration
- Entra admin center → App registrations → New registration
- Name: Okta-Integration
- Supported account types: Single tenant
- Redirect URI (platform: Web):
- https://{your-okta-domain}/oauth2/v1/authorize/callback

### 5.2 Generate Credentials
- Copied the Application (client) ID from the app's Overview page.
- Certificates & secrets → New client secret → copied the Value immediately (hidden after leaving the page).
- API permissions → added Microsoft Graph delegated permissions: User.Read, email, openid, profile → granted admin consent.

<img width="2872" height="1520" alt="Create-Okta-App-registration" src="https://github.com/user-attachments/assets/3c86119f-59da-471e-9684-24ed67f68cca" />

<img width="2870" height="1514" alt="Certificates Secrets" src="https://github.com/user-attachments/assets/f0549a3d-3f1a-41b1-87bf-d818499fa41d" />


### 5.3 Configure the IdP in Okta
- Okta Admin Console → Security → Identity Providers → Add Identity Provider → Microsoft IdP
- Note: of the many "Microsoft" catalog entries in Okta (Office 365, Dynamics, OneDrive, etc.), "Microsoft IdP" is the correct one for this step — it's the entry combining Lifecycle Management + SCIM + Inbound Federation.
- Filled in Name (Entra-Integration), Client ID, Client Secret, left default scopes.
- Result: IdP shows Active, with a generated IdP ID, Authorize URL, and Redirect URI.

<img width="2866" height="1522" alt="Configure IdP" src="https://github.com/user-attachments/assets/2da30b8a-a10a-4b0a-b678-8188ad732884" />
<img width="2826" height="1500" alt="Configure Microsoft IdP" src="https://github.com/user-attachments/assets/bd18df25-a271-4ee5-af14-c5e0e13e540d" />


### Part 6 — Okta: User Provisioning (Entra → Okta)
- This is the actual data pipeline — pulling user records (not just auth) from Entra ID into Okta.
### 6.1 Add the Provisioning Connector
- Okta Admin Console → Applications → Browse App Catalog → "Microsoft Office 365"
- This is Microsoft's official Graph-based connector for Okta, despite the name — it's used purely for identity provisioning here, not for actual Office 365 licensing.
- Catalog tags confirm this is the correct integration: Privileged Access Management, Identity Security Posture Management, Entitlement Management, WS-- -Federation, Workflow Templates, Secure Identity Integrations - Strategic, Workflows Connectors, SWA, Partial Universal Logout, Brokered Consent, SCIM.
Clicked Add Integration.

<img width="2852" height="1500" alt="Provisioning Connector" src="https://github.com/user-attachments/assets/b2e20c62-4e00-4494-aa7d-d02c02d5b520" />


### 6.2 General Settings — Tenant Name
- Application label: Microsoft Office 365
- Microsoft Tenant Name: entered as the bare tenant subdomain, no @ symbol and no .onmicrosoft.com suffix needed in some wizard versions, In this build, the working value entered was the full tenant identifier arbofelipe2003outlook.onmicrosoft.com.

<img width="2856" height="1494" alt="Tenant Name" src="https://github.com/user-attachments/assets/f5013a71-6680-427e-895d-3d916da51cfa" />


### 6.3 Sign-On Tab
- Sign-on method: Secure Web Authentication (SWA) — sufficient for lab purposes; WS-Federation would be the more "production-realistic" choice but requires additional trust configuration.
- Update application username on: initially left at Create and update — later changed to Create only after this caused outbound push errors.
- Checked "Allow administrator to consent for Advanced API access" — required for Okta to obtain the elevated Graph permissions needed for provisioning. - This triggers the Authenticate with Microsoft Account flow.

<img width="2832" height="1340" alt="Sign-On Tab" src="https://github.com/user-attachments/assets/be0f42dd-9045-471b-b067-2f3c089d95cc" />


### 6.4 Admin Consent Screen
- Authenticating surfaces a Microsoft permissions-consent screen for "Okta Microsoft Graph Client," requesting (among others):
- Read and write all users' full profiles
- Read/write all applications, all groups, all group memberships
- Read and write all directory RBAC settings
- Read organization information
- Sign in and read user profile / read and write directory data
- This is a broad, tenant-wide grant — appropriate for a lab, but worth flagging in a real design doc as something to scope down (e.g. via least-privilege Graph app permissions and admin-unit restrictions) before production use. Clicked Accept.

<img width="794" height="1600" alt="Admin Consent Screen" src="https://github.com/user-attachments/assets/4de2b5ff-f916-424e-8f4d-508cc55ec7e1" />

### 6.5 Provisioning → Integration
- Enabled API integration.
- Authenticated with a Global Admin account against the correct tenant domain.
- Confirmed with the banner: "Microsoft Office 365 was verified successfully!" and "Microsoft Office 365's API is authenticated."

<img width="2040" height="1374" alt="Provisioning Integration" src="https://github.com/user-attachments/assets/0a4ed711-a4c9-46b8-b97b-8e81d630180f" />


### 6.6 Provisioning → To App
- Enabled all three:
- Create Users
- Update User Attributes
- Deactivate Users (with Microsoft Office 365 user status on deactivation set to Block sign-in)

<img width="1520" height="948" alt="Provisioning To App" src="https://github.com/user-attachments/assets/4b4f54ae-5dac-4094-9d3e-1fbb72582bc0" />


### 6.7 Provisioning → To Okta (Import Settings)
- Provisioning type: Profile Sync (not User Sync — User Sync is explicitly incompatible with Entra Connect/AADConnect).
- Checked "Import unlicensed users" — required, since synced AD users typically have no M365 license assigned, and without this setting the import silently only finds licensed accounts.


<img width="1402" height="888" alt="Provisioning To Okta1" src="https://github.com/user-attachments/assets/6c940753-7662-4305-a078-c9bf2358f54f" />

<img width="1418" height="360" alt="Provisioning To Okta2" src="https://github.com/user-attachments/assets/37f2437e-ec2e-4e8e-88cb-c4571c9f625d" />


### 6.8 Assign the Scoped Group
- Assignments tab → Assign to Groups → searched "okta" → assigned Okta Synced Users.

<img width="2828" height="1490" alt="Assign the Scoped Group" src="https://github.com/user-attachments/assets/2e809999-6a1e-432e-901c-1a9d3de8e1df" />


### 6.9 Run the Import
- Provisioning → Import → Import Now
- Reviewed Import Results: all three users showed "NO Okta user matches found" → "NEW Okta user" (expected, first-time creation).
- Clicked Confirm Assignments (3) to finalize creation of the three Okta user objects.

<img width="1742" height="712" alt="Run Import" src="https://github.com/user-attachments/assets/9a3fed10-29b2-478f-8f23-5349e1495158" />


### 6.10 Verify in Okta's Local Group
- Okta Admin Console → Directory → Groups → Okta Synced Users (the Okta-side group from 6.8) → People tab.
- Confirmed all three users present: OktaUser1/2/3, each showing status Staged.
- Staged ≠ Active. A "Staged" status means the user object exists in Okta but has not yet been activated

<img width="2080" height="1394" alt="Verify" src="https://github.com/user-attachments/assets/f4bee399-9cf1-464b-9201-df903d53b569" />


### 6.11 Configure Attribute Mapping (Office 365 → Okta)
- User objects existed at this point, but Department/Title/Company were still blank on the Okta side — provisioning creates the user record, but attribute values only flow once explicit mappings exist.
- On the app's Provisioning → To Okta tab, scrolled to "Microsoft Office 365 Attribute Mappings." This section has two relevant controls:
- Go to Profile Editor — opens the actual mapping editor (below).

<img width="1300" height="1074" alt="Configure Attribute Mapping" src="https://github.com/user-attachments/assets/44f0ed85-c8e1-4939-9126-5d1e73b85cd7" />

- Force Sync — re-pulls attribute values for already-created users using whatever mappings currently exist; needed after any mapping change, since existing users don't retroactively pick up new mappings on their own.

<img width="1418" height="1150" alt="Force Sync" src="https://github.com/user-attachments/assets/438f8c34-80f5-41a1-9384-0f1280cdf02b" />

- Clicked Go to Profile Editor, which opens a two-column view: Microsoft Office 365 User Profile (source, labeled appuser) on the left, mapping into the Okta User Profile (user) on the right.
- For each target Okta attribute, set the source expression using the field's own dropdown/autocomplete rather than free-typing it:
- appuser.Title → mapped to Okta's title
- appuser.Company → mapped to Okta's organization
- appuser.Department → mapped to Okta's department

<img width="1166" height="656" alt="appuser Title" src="https://github.com/user-attachments/assets/df01941a-fc0f-459d-9f5e-24c79b4ea1a6" />
<img width="1178" height="476" alt="company:department" src="https://github.com/user-attachments/assets/4dc31d94-cca2-4598-899e-49180c684c2d" />


- Saved the mappings, went back to Provisioning → To Okta, and clicked Force Sync to push the corrected values onto the three already-existing users.
- Confirmed in the OktaUser1 profile that values now resolved correctly (e.g. for OktaUser1: Department: Analyst, company: Okta, job title: Okta Analyst).

<img width="1392" height="1220" alt="User attributes" src="https://github.com/user-attachments/assets/68492f03-2f82-480f-9512-6f0d0ba52d7c" />
<img width="1388" height="632" alt="User attributes2" src="https://github.com/user-attachments/assets/19e15840-c05d-48db-b7aa-9a443d3ddcb1" />


### Part 7 — Okta: App Assignment (SSO "Front Door")
- To validate the pipeline actually delivers app access, added a simple demo app rather than a real SaaS product.
### 7.1 Add a Bookmark App
- Applications → Applications → Browse App Catalog → "Bookmark App" → Add Integration.
- Application label: Northstar Project
- URL: https://www.google.com (placeholder destination for demo purposes — a Bookmark App just redirects to any URL after successful sign-in).
Saved as Done.

<img width="2740" height="1226" alt="Bookmark App integration" src="https://github.com/user-attachments/assets/f28c27c7-fad6-4184-8bf4-9bfaead56707" />

<img width="1818" height="1058" alt="Add Bookmark App" src="https://github.com/user-attachments/assets/5b9c54a1-feef-4e65-b8db-856df72211e1" />


### 7.2 Assign the App to the Scoped Group
- On the new app's Assignments tab → Assign → Assign to Groups.
- Assigned Okta Synced Users — all three users inherited access automatically as Group assignments, rather than assigning each user individually.

<img width="1862" height="1250" alt="Assign to scoped group" src="https://github.com/user-attachments/assets/d3f5eb67-a623-43d4-ad48-ef9b312a8385" />

<img width="1776" height="1202" alt="App scoped people" src="https://github.com/user-attachments/assets/0ae3d374-7d5f-4f82-b2de-8d4594d01ec9" />

### Part 8 — Okta: Authentication (MFA) Policy
- Added policy enforcement on top of the plain SSO tile.
### 8.1 Create the Policy Rule
- Security → Authentication Policies → selected/created a policy for the app (Okta Project - MFA Policy, attached via the app's Application tab → Switch policy).
- Add Rule, named Northstar Project Rule.
- Conditions:
- User's group membership includes at least one of: Okta Synced Users
- User is: Any user
- Device state: Any
- Device assurance policy: No policy
- Device platform: Any platform
**- OBS: Enterprise environments would have a more granular control over these settings but for the sake of the project only the group mentioned is being assigned to the policy**

<img width="1706" height="1456" alt="Add Rule" src="https://github.com/user-attachments/assets/78c4e0c7-d72d-4666-99d0-06a04ba3a1b4" />

### 8.2 Access Requirements
- Access is: Allowed after successful authentication
- User must authenticate with: Password / IdP + Another factor
- Possession factor constraints:
- ❌ Phishing resistant — left unchecked (would require FIDO2/hardware keys, which weren't set up; checking this would have locked test users out entirely)
- ❌ Hardware protected — unchecked
- ✅ Require user interaction → Any interaction (covers an Okta Verify push approval or similar)
- Authentication methods: Allow any method that can be used to meet the requirement
- Org's authenticators satisfying this: Password AND (Okta Verify - FastPass or Okta Verify - TOTP)
- Re-authentication frequency: both password and other-factor prompts set to re-trigger when it's been over 1 hour since the user's last activity on the active Okta global session (rather than "every time" or "never while session exists").

<img width="1294" height="1376" alt="Access Requirements" src="https://github.com/user-attachments/assets/ee435893-d4ba-4bde-a705-dfbac0b48265" />

<img width="1310" height="848" alt="Access Requirements2" src="https://github.com/user-attachments/assets/fe878ab2-b2a9-494b-a4e7-62da0189e3ee" />


# 8.3 Result
- Policy created and enabled at Priority 1, above the org's default Catch-all Rule (Priority 2, which allows access with any 2 factor types for everything not matched by a more specific rule).
- Added the **Northstar Project App** to the policy

<img width="2316" height="1316" alt="Result" src="https://github.com/user-attachments/assets/970730c6-bf8c-42c4-bb02-7fb627836961" />

<img width="2274" height="806" alt="Result2" src="https://github.com/user-attachments/assets/01d3c31e-aea4-46e0-a207-fc0b49fccff9" />


### Part 9 — Okta: Password Policy
- Added a dedicated password policy scoped to the same group, separate from Okta's org-wide default.
### 9.1 Create the Policy
- Security → Authenticators (confirmed Okta Verify was already added/available as an authenticator before continuing).

<img width="1188" height="1126" alt="Add Authenticator" src="https://github.com/user-attachments/assets/53384f35-b9b6-4217-9d9e-dc037739e335" />


- Security → Authentication Policies → (Password policies area) → Add New Password Policy.
- Policy name: Northstar Project - Password Policy
- Policy description: "Password policy for synced Entra ID users"
- Add group: Northstar Synced Users
- Authentication Providers → Applies to: Okta

### 9.2 Password Requirements
- Minimum length: 12 characters
- Complexity requirements, all checked:
- Lowercase letter, Uppercase letter, Number (0–9), Symbol
- Does not contain part of username, first name, or last name
- Left "Maximum 2 consecutive repeating characters" unchecked.

<img width="1532" height="1158" alt="Passord Requirments" src="https://github.com/user-attachments/assets/492bd999-7251-4615-a25a-61f847a78c48" />

### 9.3 Policy Rule
- Add Rule → Northstar Password Policy Rule
- If User's IP is: Anywhere
- Then users can perform self-service: Password change, Password reset, Unlock account — all enabled
- Recovery authenticators / Access control: left on "Authentication policy" (uses the Okta account management policy for controlling reset/unlock conditions, rather than the legacy per-rule setting).

<img width="938" height="904" alt="Image 9-8-26 at 22 02" src="https://github.com/user-attachments/assets/9b597485-fc56-421f-960e-209e3bcb3ab2" />

### 9.4 Result
- Northstar Project - Password Policy shown as Active, listed above the org's Default Policy, scoped specifically to the Okta Synced Users group.

<img width="1708" height="1096" alt="Password Result" src="https://github.com/user-attachments/assets/a2af897e-c52a-4f94-9c8c-32a157a8b7b8" />


### Part 10 — End-to-End Sign-In Test
- With provisioning, attribute mapping, app assignment, MFA policy, and password policy all in place, ran a full sign-in test as a real end user.
### 10.1 Activate the User
- Directory → People → NorthstarUser3 → More Actions → Set Password & Activate (equivalent: Reset Password, which generates a temporary password immediately, e.g. 6vQ9T?%riC!2).
- Result: user status moved from Staged → Pending user action.

<img width="2864" height="1420" alt="Actice User" src="https://github.com/user-attachments/assets/2516aaf6-d983-4ec9-a13f-63929ceaa5af" />

### 10.2 First Sign-In as the End User
- Navigated to the Okta end-user sign-in page and signed in as northstaruser1@ehsegvichgmail.onmicrosoft.com with the temporary password.
- Immediately prompted: "Your password has expired" — required to set a new password meeting the Part 9 policy (12+ chars, mixed case, number, symbol, no name fragments).
- After successfully changing the password, prompted to "Set up security methods" — Okta Verify shown as Required now (this is the MFA policy from Part 8 kicking in on first login).

<img width="1440" height="1484" alt="End User Sign-In" src="https://github.com/user-attachments/assets/591eea33-640e-42c3-8118-ec657d736f48" />

<img width="1044" height="1138" alt="Okta Verify" src="https://github.com/user-attachments/assets/da3927d0-9c90-4c4c-8595-edc48b464869" />

### 10.3 Reach the Dashboard and Launch the App
- After completing Okta Verify enrollment, landed on the end-user My Apps dashboard, signed in as OktaUser3.
- The Okta Project tile (the Bookmark App from Part 7) was visible under "Work" — confirming the group-based assignment worked correctly.
- Clicked the tile → redirected successfully to google.com, confirming the SSO "front door" flow works end-to-end: AD-provisioned identity → Entra sync → Okta provisioning → policy-enforced authentication → app launch, all without any manual per-user app configuration.

<img width="2870" height="1498" alt="User Dashboard" src="https://github.com/user-attachments/assets/8f76a8a0-46b1-4b1c-9189-6a654de8eda9" />

<img width="2190" height="1070" alt="Screenshot 2026-10-10 at 3 39 45 PM" src="https://github.com/user-attachments/assets/cb143441-4bc4-42d0-9898-6a06982d12e8" />

# Lessons Learned / Known Issues
- These are genuine architectural constraints worth documenting, not just lab friction:
### 1. AD-mastered objects cannot be written to from downstream systems
- Attempting to push attribute or username updates back to Office 365/Entra for AD-synced users fails with:
- HTTP 400 — Request_BadRequest
- "Unable to update the specified properties for on-premises mastered
- Directory Sync objects or objects currently undergoing migration."
- This is a hard platform rule, not a misconfiguration: once AD is authoritative for a user, nothing downstream (Entra, Okta, any SCIM consumer) is allowed to write to those attributes. This directly validates the "AD as source of truth" architecture decision — and means any provisioning connector must be configured read-only inbound for these users.
- Fix applied: disabled "Update User Attributes" (To App) and changed "Update application username on" (Sign On tab) from Create and update to Create only — both are outbound-push triggers that must stay off for AD-mastered users.
### 2. Stale provisioning tasks don't self-clear
- Even after fixing the setting that caused a push error, the failed job remains queued and keeps surfacing the same error until manually dismissed via Okta Admin Console → Tasks page.
### 3. Attribute mapping expressions are case-sensitive and must match exact Graph property names
- Okta's Profile Editor mapping expressions (e.g. appuser.department) must exactly match Microsoft Graph's property casing — the Entra admin UI's display labels don't always match:
- Entra UI Label	Correct Graph property name
- Job title	jobTitle
- Company name	companyName
- Department	department
- Common mistakes made during this build: appuser.CompanyName (wrong name entirely — should be companyName), appuser.Department (wrong casing), "appuser.jobTitle" (wrapped in quotes, which Okta evaluates as a literal string rather than an expression).
### 4. Multiple duplicate app instances can be created accidentally
- Retrying "Add Integration" from the catalog can result in multiple separate Microsoft Office 365 User app profiles in Okta's Profile Editor (e.g. office365, office365_6i83i76, office365_7doe7g0), each with independent mapping configuration. Always verify which app instance is actually assigned/active (check the Assignments tab) before editing mappings — editing the wrong instance wastes time with no visible effect.
### 5. Attribute mapping — resolved
- Earlier attempts to map Department/Title/Company failed due to a combination of wrong casing, wrong property names, and quoted literal strings (see #3 above). The working fix is documented in full under Part 6.11. In short: expressions had to be selected via the mapping field's own dropdown/autocomplete rather than free-typed — manual typing repeatedly introduced casing errors or invalid property names that Okta's UI doesn't validate until you actually try the suggestion list.
What's Working End-to-End
- ✅ AD → Entra ID sync (Entra Connect)
- ✅ Dynamic group scoping in Entra ID
- ✅ Okta authentication federation (Identity Provider)
- ✅ Okta user provisioning/creation from Entra ID (Create Users)
- ✅ Attribute mapping (Department/Title/Company) resolved and verified via live preview
- ✅ Deactivate Users provisioning enabled (deprovisioning cascade — configuration in place, propagation not yet tested)
- ✅ App assigned via group (Bookmark App demo, "Okta Project")
- ✅ Authentication (MFA) policy enforcing Password + Okta Verify, scoped to the synced-user group
- ✅ Dedicated password policy (12-char complexity) scoped to the synced-user group
- ✅ Full end-to-end sign-in tested successfully: AD-provisioned user → Entra sync → Okta provisioning → forced password reset → MFA enrollment → dashboard → app launch
