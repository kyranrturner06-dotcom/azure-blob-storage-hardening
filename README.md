# Azure-Blob-Storage-Hardening

# <ins>**Introduction**</ins>

This project will manage; encryption, access, and logging of an Azure Blob Storage Account. In regards to encryption, Key Vault will be used to store the cryptographic keys for the storage account and will also be where I alter configurations to allow for Customer Managed Keys, rather than Microsoft Managed Keys. Secondly, A Log Analytics workspace will be set up to allow for logging of activities within the storage account, allowing for a clear audit trail. Finally on the access front, a misconfigured, dangerous SAS token will be rectified and a Private Endpoint will be set up to only allow access from within the VNet, not the public internet.

# <ins>**Environment Set-Up**</ins>

This lab requires four distinct resources to be created. Firstly I made a resource group (Lab3-RG) to house the storage account and other services. Next, the storage account itself was created under a Standard Blob Storage Account set up in region Norway East. Then came the Key Vault, and finally the Log Analytics Workspace. See Image Below.

<img width="1574" height="483" alt="Screenshot 2026-08-13 112515" src="https://github.com/user-attachments/assets/9b4b6729-7fa3-4bbe-9ec7-a50bbad46746" />

# <ins>**CMK in Key Vault**</ins>

By default, storage accounts are set to MMK (Microsoft Managed Keys), this isn't actually a bad thing as MMK is secure, however it does not allow as much personal freedom to manage the key. For instance you have no control over the key rotation schedule, no power to disable the key which in turn revokes access, and can dissatisfy compliance in certain situations where you are required to have full control (for instance government related regulations), so I will be changing storage account configuration to CMK.

_Image 2: (before changes)_
<img width="984" height="382" alt="Screenshot 2026-08-13 113150" src="https://github.com/user-attachments/assets/de37f6ba-9d01-4d2a-8377-06a812440c6e" />

**Creating a Key Within Key Vault**

Before I can switch the Storage Account to CMK, I need to prepare Key Vault to be able to handle it. To do this, I firstly need to generate a Key inside Key vault which will be the Key which I can manage. To do this I went to Key Vault > Objects > Keys, clicked generate/import, and finally configured the key settings to be: Generate (not import), an RSA Key Type (Asymmetric Encryption), and a 2048 Key Size.

I ran into an issue however, I did not have sufficient RBAC permissions to generate/view the key. This meant I had to go to the IAM panel within the key vault, and assign myself the Key Vault Crypto Officer role, allowing me to perform any action on the keys within the key vault. Finally, I had to go to Key Vault > Settings > Properties to enable Purge Protection. In turn meaning if the Key was to be Deleted, it could be recovered within a strict time period, this is a requirement to activate CMK.

_Image Set 3:_
<img width="1443" height="616" alt="Screenshot 2026-08-13 113410" src="https://github.com/user-attachments/assets/5ad31bc1-99f5-491b-8d57-a4e37ef756e7" />
<img width="1097" height="430" alt="Screenshot 2026-08-13 113643" src="https://github.com/user-attachments/assets/27d2d0ed-096a-4061-9356-dd28357cc884" />

**Converting Storage Account to CMK**

To achieve this, i went to security + networking > Encryption, all within the storage account. I then changed the settings to be; Encryption Type = CMK, Encryption Key = Select from key vault (i then selected the key vault i had created, along with the key i previously made inside it), and Identity type = System assigned. That last option meant a 1:1 Managed Identity was created on top of the storage account, which is crucial as the storage account will need to be able to authenticate to key vault on behalf of itself in order to access secrets. 

_Image 4:_
<img width="1162" height="626" alt="Screenshot 2026-08-13 121910" src="https://github.com/user-attachments/assets/50c3a854-2d3f-493c-972e-b794fb55e9b2" />


**Key Vault RBAC on the Managed Identity**

Curiously, the previous step wouldn't completely execute. I received an error message saying CMK could not be activated as the selected Key Vault had not given sufficient access roles to the Storage Account Managed Identity. In response, I went back to the Key Vault IAM tab, clicked 'Add Role Assignment', selected 'Key Vault Crypto Service Encryption User', then under the members tab I selected managed identity and somehow the storage account was there despite it previously saying the provisioning failed. I should note, I chose Key Vault Crypto Service Encryption User for the managed identity to allow it to read keys and wrap/unwrap encrypted items.

After some research, I learned this was a strange quirk where the storage account successfully provisions a managed identity, then searches to see if itself has access to the key vault, and finally says the process failed because it didn't have access. To continue onward, I simply assigned the Managed Identity the previously named access role, went back to the storage Encryption Settings, and completed the CMK conversion by selecting the same settings as before when it initially half failed.

_Image Set 5:_                       
<img width="1079" height="459" alt="Screenshot 2026-08-13 122622" src="https://github.com/user-attachments/assets/37c744b8-81b4-40ab-ae7f-0b8c9f6e02eb" />
<img width="932" height="407" alt="image" src="https://github.com/user-attachments/assets/b5532936-01de-47d4-867b-09b7f37f7725" />

# <ins>**Enabling Diagnostic Logging**</ins>








