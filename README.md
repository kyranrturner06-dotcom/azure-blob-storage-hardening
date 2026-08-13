# Azure-Blob-Storage-Hardening

# <ins>**Introduction**</ins>

This project will manage; encryption, access, and logging of an Azure Blob Storage Account. In regards to encryption, Key Vault will be used to store the cryptographic keys for the storage account and will also be where I alter configurations to allow for Customer Managed Keys, rather than Microsoft Managed Keys. Secondly, A Log Analytics workspace will be set up to allow for logging of activities within the storage account, making for a clear view of events. Finally on the access front, a misconfigured, dangerous SAS token will be rectified and a Private Endpoint will be set up to only allow access from within the VNet, not the public internet.

# <ins>**Environment Set-Up**</ins>

This lab requires four distinct resources to be created. Firstly I made a resource group (Lab3-RG) to house the storage account and other services. Next, the storage account itself was created under a Standard Blob Storage Account set up in region Norway East. Then came the Key Vault, and finally the Log Analytics Workspace. See Image Below.

<img width="1574" height="483" alt="Screenshot 2026-08-13 112515" src="https://github.com/user-attachments/assets/9b4b6729-7fa3-4bbe-9ec7-a50bbad46746" />

# <ins>**CMK in Key Vault**</ins>

By default, storage accounts are set to MMK (Microsoft Managed Keys), this isn't actually a bad thing as MMK is secure, however it does not allow as much personal freedom to manage the key. For instance you have no control over the key rotation schedule, no power to disable the key which in turn revokes access, and can dissatisfy compliance in certain situations where you are required to have full control (for instance government related regulations), so I will be changing storage account configuration to CMK.

_Image 2: (before changes)_
<img width="984" height="382" alt="Screenshot 2026-08-13 113150" src="https://github.com/user-attachments/assets/de37f6ba-9d01-4d2a-8377-06a812440c6e" />

**Creating a Key Within Key Vault:**

Before I can switch the Storage Account to CMK, I need to prepare Key Vault to be able to handle it. To do this, I firstly need to generate a Key inside Key vault which will be the Key which I can manage. To do this I went to Key Vault > Objects > Keys, clicked generate/import, and finally configured the key settings to be: Generate (not import), an RSA Key Type (Asymmetric Encryption), and a 2048 Key Size.

I ran into an issue however, I did not have sufficient RBAC permissions to generate/view the key. This meant I had to go to the IAM panel within the key vault, and assign myself the Key Vault Crypto Officer role, allowing me to perform any action on the keys within the key vault. Finally, I had to go to Key Vault > Settings > Properties to enable Purge Protection. In turn meaning if the Key was to be Deleted, it could be recovered within a strict time period, this is a requirement to activate CMK.

_Image Set 3:_
<img width="1443" height="616" alt="Screenshot 2026-08-13 113410" src="https://github.com/user-attachments/assets/5ad31bc1-99f5-491b-8d57-a4e37ef756e7" />
<img width="1097" height="430" alt="Screenshot 2026-08-13 113643" src="https://github.com/user-attachments/assets/27d2d0ed-096a-4061-9356-dd28357cc884" />

**Converting Storage Account to CMK:**

To achieve this, i went to security + networking > Encryption, all within the storage account. I then changed the settings to be; Encryption Type = CMK, Encryption Key = Select from key vault (i then selected the key vault i had created, along with the key i previously made inside it), and Identity type = System assigned. That last option meant a 1:1 Managed Identity was created on top of the storage account, which is crucial as the storage account will need to be able to authenticate to key vault on behalf of itself in order to access secrets. 

_Image 4:_
<img width="1162" height="626" alt="Screenshot 2026-08-13 121910" src="https://github.com/user-attachments/assets/50c3a854-2d3f-493c-972e-b794fb55e9b2" />


**Key Vault RBAC on the Managed Identity:**

Curiously, the previous step wouldn't completely execute. I received an error message saying CMK could not be activated as the selected Key Vault had not given sufficient access roles to the Storage Account Managed Identity. In response, I went back to the Key Vault IAM tab, clicked 'Add Role Assignment', selected 'Key Vault Crypto Service Encryption User', then under the members tab I selected managed identity and somehow the storage account was there despite it previously saying the provisioning failed. I should note, I chose Key Vault Crypto Service Encryption User for the managed identity to allow it to read keys and wrap/unwrap encrypted items.

After some research, I learned this was a strange quirk where the storage account successfully provisions a managed identity, then searches to see if itself has access to the key vault, and finally says the process failed because it didn't have access. To continue onward, I simply assigned the Managed Identity the previously named access role, went back to the storage Encryption Settings, and completed the CMK conversion by selecting the same settings as before when it initially half failed.

_Image Set 5:_                       
<img width="1079" height="459" alt="Screenshot 2026-08-13 122622" src="https://github.com/user-attachments/assets/37c744b8-81b4-40ab-ae7f-0b8c9f6e02eb" />
<img width="932" height="407" alt="image" src="https://github.com/user-attachments/assets/b5532936-01de-47d4-867b-09b7f37f7725" />

# <ins>**Enabling Diagnostic Logging**</ins>

Visibility and Detection is a major part of security, enabling logging provides insight into what happened if an incident occurs, it's also important in regards to digital forensics. As a result, I will be configuring Storage Blob Diagnostics Logs to get sent to Log Analytics Workspace where they can be views and queried with KQL.

To do this, I went to Storage Account > Monitoring > Diagnostics Settings, which brought up a selection of the different storage account sub-categories (blob, queue, table, file.) I specifically selected the Blob option to then click 'add diagnostics setting' which brought up a wizard where I could enable the collection of certain logs, and select where to send them.

_Image 6:_                       
<img width="1591" height="468" alt="Screenshot 2026-08-13 131238" src="https://github.com/user-attachments/assets/3e510310-3c61-4fe0-af1d-76ae8cfbd392" />

**Diagnostics Settings:**

- I named the setting 'blob-audit-logs'
- For categories I manually clicked 'Storage Read' (collects who read a blob), 'Storage Write' (collects who created/modified a blob), and 'Storage Delete' (collects who deleted a blob, which is very important).
- Destination Details was configured to send to Log Analytics Workspace, specifically the Log Analytics Workspace I created previously.

_Image Set 7:_                       
<img width="949" height="611" alt="Screenshot 2026-08-13 141131" src="https://github.com/user-attachments/assets/357c78ad-1191-4380-bc5c-a64ce805ea6a" />
<img width="747" height="192" alt="image" src="https://github.com/user-attachments/assets/01dc8d4f-9393-4e43-9e18-d66056d9460c" />

**KQL Query Tests:**

To ensure the diagnostic data was correctly being ingested by Log Analytics, I ran a KQL query to see if results registered. This meant I needed something to get results from, so for that purpose I created a container within the storage account, uploaded a file, then deleted the file placed in the container. This should trigger Storage Write and Storage Delete rules, creating 'PutBlob' and 'DeleteBlob' entries.

The KQL Query:
- The 'Where' line determines what rows I want to filter by, here it says 'return rows which have PutBlob or DeleteBlob in the OperationName column'
- The 'Project' line determines what columns I want returned, so the specified columns will have their results returned as long as PutBlob or DeleteBlob is in the same row.
- The 'Order By' line simply defines how the results should be sorted, in this case it is by their date, and set to descending.
- Finally, the | operator links all of the commands together, so they act with context to each other.

_Image 8:_   
<img width="1113" height="582" alt="Screenshot 2026-08-13 154146" src="https://github.com/user-attachments/assets/15bd14cd-e52c-46df-98fc-8a52aa5bb804" />

# <ins>**Over-Privileged SAS Token**</ins>

I purposely created an SAS token with too many permissions. To start, the SAS token expires in a year, which is way too long for something which is meant to be ephemeral. It is scoped to too many services and resources, breaking the concept of least-privilege, you can see this as all options for Allowed services & Allowed resource types are ticked. This isn't to mention that whoever gets hold of this SAS URL will have extremely high permissions, they can do anything they like to the Storage account.

If this was to be leaked, which could happen via it being logged/messaged/committed in a repo, then the entire account would be compromised and there would be no way to revoke the permissions from a specific SAS, meaning you would have to regenerate the accounts storage keys to make the SAS void, by doing this you would also make all other SAS tokens signed with the same key void as well which will lock legitimate users out, impacting availability.

_Image Set 9: (Cryptographic Signature [Sig=] I have redacted so access can't be achieved, this is a repo so that would be a leak.)_
<img width="1885" height="852" alt="Screenshot 2026-08-13 160742" src="https://github.com/user-attachments/assets/b7675538-e2d4-4083-95ee-e3fb2d9163d8" />
<img width="1648" height="362" alt="Screenshot 2026-08-13 161704" src="https://github.com/user-attachments/assets/30510a85-1f7f-47f6-be43-bfc1118dbecb" />

**Correcting the SAS Token:**

Fixing the issues, I started with the activation time frame, I changed it from a year to an hour, SAS tokens are not meant to be long term. I then only allowed it to act upon the Blob Service, and specifically the Object resource inside the Blob. Then, set it to a read only token so the receiver can not make any modifications. On that topic, I also removed any Blob index permissions so they can not work with/modify index tags. Finally I added a specific IP address range to ensure more security on who can access the resource, even with the token.

_Image 10:_
<img width="1007" height="715" alt="image" src="https://github.com/user-attachments/assets/390720ba-c82e-4900-bb36-b56d5adad619" />

# <ins>**Private Endpoint Set-Up**</ins>

Finally, I need to secure network access to the storage account, as so far the focus has been on identity based access. To do this I will restrict access to the storage account from the internet, and access will only be granted via a private endpoint pairing a VNet to the Storage Account. A requirement for this is that I set up a VNet within the resource group, which was done easily by searching 'Virtual Network' in the portal and configuring it to the resource group. It was only a simple VNet containing one Subnet as I do not need anything complex.

**Private Endpoint:**

To create the private endpoint, I went to Storage Account > security + networking > Networking > Private Endpoints, the clicked 'Create'. Now onto the configuration, for the basics, I selected my resource group, which then auto-filled the region subscription. I also gave it the 'Private_Endpoint_Lab3' name. Under the resource tab, I set the Link type to Storage Accounts, selected my resource group, then selected the 'lab3storagekt' storage account. I put the Target Sub-resource as blob. For the Virtual Network tab I selected the VNet I had just created. DNS and Tags I left as default.

With all of this done, I finally needed to turn off public internet access, which was done in the same 'Networking' panel within the storage account (Storage Account > security + networking > Networking). All I had to do was click on 'public network access' and set it to 'disable'.

_Image Set 11:_                                   
<img width="653" height="810" alt="Screenshot 2026-08-13 173812" src="https://github.com/user-attachments/assets/b4aaf980-5c43-4fa2-b034-74a46d1d4854" />
<img width="1151" height="270" alt="Screenshot 2026-08-13 174337" src="https://github.com/user-attachments/assets/1472d6e9-723a-4de7-973e-5fbfe576e93d" />
<img width="1214" height="569" alt="Screenshot 2026-08-13 175122" src="https://github.com/user-attachments/assets/48d48e48-5010-47cf-bc2e-7d324146806a" />

**Testing Private Endpoint Functionality:**

I was initially going to test this by getting a blob URL from a container within the private-linked storage account, however i realised I could not even do that because viewing was restricted to only traffic coming from within the VNet, so my browser was blocked. This is equally validating that the Private Endpoint works, so I was happy with the result and have attached a screenshot of the error message I received.

In contrast, to test the functionality from within the VNet, I would need to deploy a VM inside the VNet to then SSH/RDP into the VM. This would mean when I try to access the container files in the storage account from the VM, access would be granted as the VM is inside the VNet. I will not be doing that in this lab however as setting up a VM is outside of its scope.

_Image 12:_
<img width="1430" height="563" alt="Screenshot 2026-08-13 175014" src="https://github.com/user-attachments/assets/4a24b61f-2bc7-457d-86b9-d68acc315a77" />

# <ins>**Conclusion**</ins>

That marks the end of this Storage Account Hardening project. To summarise, I began by changing the Storage Account Encryption settings from MMK (the default) to CMK. This allowed greater control over the key and would put the storage account into compliance with laws related to financial/government data. CMK gives you more control over your encrypted data at rest. Next, I set up a form of visibility over actions took within the storage account through routing diagnostic logs to a log analytics workspace. This gives insight into what is happening, so you can potentially catch and mitigate a threat before more significant damage is done by just querying log data. For instance an insider threat could be caught deleting/modifying data, and in turn could have their permissions revoked in EntraID/RBAC as a result of log analysis. Finally I secured the access to the storage account by setting up a Private Endpoint linked to a VNet, later disabling internet access so the storage account is only accessible through the VNet, in turn wiping out a large amount of attackers who at least initially will come from the internet.

In terms of mistakes, i had trouble with the Log Analytics ingestion of data. I initially set the diagnostics setting to 'audit', then manually selected Storage Read/Storage Write/Storage Delete, and later selected 'AllLogs', but each setting did not successfully catch the deletion of the container. After some attempts, i realised i had been deleting the container itself and not the blob file inside, so the deletion event was not getting caught. Once I found this out though, I switched the setting back to the manually selected trio since 'AllLogs' was far too broad and would ingest too much unwanted data. 

While not a mistake, I also learned from the half provisioning issue I had, where the managed identity was deployed, but the rest of the settings were not carried out. Another notable event was the requirement to provision two roles when setting up CMK. The user setting up key vault must have the crypto officer role, and then another role is needed for the managed identity, being crypto service encryption user. I feel like I got a better grasp of how an SAS token could be hastily over-provisioned, since by default azure has some boxes already ticked, but having a short time span and minimal resources/privileges ticked really is important for least-privilege and security as a whole.

One quick note, in the real world MMK is not bad, it removes operational overhead which could outweigh the benefits of the increased control which CMK brings, and also logging diagnostic data is useless unless someone is there to pay attention to it. 

In the end, I gained more hands on experience with the concepts I leant about in AZ-500/SC-500, like Key Vault, Private Endpoints, VNets, SAS Tokens, and more.












