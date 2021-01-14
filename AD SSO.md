# ­­­­AD SSO Deployment for AKSHCI

# Overview

Without Active Directory Authentication, users have to rely on certificate based kubeconfig when connecting to the api-server via kubectl (command line tool). This kubeconfig contains secrets such as private keys and certificates that need to be carefully distributed which represents a significant security risk.

In lieu of using certificate based kubeconfig, an alternate and secure way is to use Active Directory (AD) logged in credentials (SSO) to connect to the api-server. AD integration with AKS-HCI lets a user on a windows domain joined machine connect to the api-server (via kubectl) using their SSO credentials. This removes the need to manage and distribute certificate based kubeconfigs that contain raw credential information.

AD integration uses AD kubeconfigs, these are distinct from certificate based kubeconfigs and don&#39;t contain any secrets.

The certificate based kubeconfig can be used for backup purposes such as troubleshooting if there are any issues with connecting with Active Directory credentials.

Another security benefit with AD integration is that the users and groups are stored as [SIDs](https://docs.microsoft.com/en-us/troubleshoot/windows-server/identity/security-identifiers-in-windows), unlike group names, [SIDs](https://docs.microsoft.com/en-us/troubleshoot/windows-server/identity/security-identifiers-in-windows) are immutable and unique and pose no naming conflict.

_ **Note** __: We are currently rolling out AD SSO support for managing target clusters only._

# Concepts

There are several steps involved in setting up Active Directory as the identity provider and enabling SSO via kubectl.

- Before beginning, create the AD account for the api-server and then create [keytab](https://web.mit.edu/kerberos/krb5-devel/doc/basic/keytab_def.html) file associated with the account, refer to below section for creating AD account and generating the keytab file
- Next, use the [keytab](https://web.mit.edu/kerberos/krb5-devel/doc/basic/keytab_def.html) file to install AD Auth on the K8 cluster, a default RBAC configuration is automatically created as part of this step
- Use helper tool below to convert group names to SIDs and vice-versa while creating or editing RBAC configurations

Before starting, ensure the latest PowerShell modules have been downloaded to the windows server machine. Refer **this link** for guidance on downloading the PowerShell modules. Follow **this link** to set up mgmt. cluster.

# **Part A: Use keytab file to install AD Auth**

_Step 1: Install target cluster with &quot;-enableADAuth&quot; option_

_New-AksHciCluster -clusterName_ _ **mynewcluster1** _ _-kubernetesVersion v1.18.8 -controlPlaneNodeCount 1 -linuxNodeCount 1 -windowsNodeCount 0 -controlPlaneVmSize Standard\_A2\_v2 -loadBalancerVmSize Standard\_A2\_v2 -linuxNodeVmSize Standard\_K8S3\_v1 -windowsNodeVmSize Standard\_K8S3\_v1 -__ **enableADAuth** _

For details on creating the target cluster, please refer to this link

Few things to note before proceeding to the next steps

1. The keytab file is available before continuing
2. The keytab file is generated for specific SPN (service principal name), this SPN is corresponding to the api-server AD account for the target cluster, ensure the same SPN is used in the next command
3. One api-server AD account is created per target cluster
4. **Note, the keytab should be named as current.keytab file**

_Step 2: Install AD Authentication_

_Install-AksHciAdAuth -clusterName mynewcluster1 -keytab .\current.keytab -SPN k8s/apiserver@BAKSDOM.NTTEST.MICROSOFT.COM -adminUser BAKSDOM\bugbash_

Few things to note before proceeding to the next steps

1. The keytab file is named exactly as **current.keytab**
2. Replace the SPN corresponding to your environment
3. _-adminUser_ creates a corresponding role binding for the specified AD group with cluster admin privileges, replace &quot;bugbash&quot; with AD group or user corresponding to your environment
4. Pass the admin username or group name in SID format if the **cluster host is not domain joined, refer example below**

_Install-AksHciAdAuth -clusterName mynewcluster1 -keytab .\current.keytab -SPN k8s/apiserver@BAKSDOM.NTTEST.MICROSOFT.COM -adminuserSID (or admingroupSID)_

Refer to this section below on how to get the SID

_Step 3: Test to make sure AD webhook is running and keytab is stored as K8 secret_

1. Generate kubeconfig (note this is the cert based kubeconfig), we will use this to connect to the cluster as local host, use command _Get-AksHciCredential mynewCluster1_
2. Run kubectl on the server you are connected using the certificate based kubeconfig generated in the prior step and check for the AD webhook deployment, it is of the form &quot;ad-auth-webhook-XXXX..&quot;,
_kubectl.exe get pods -n=kube-system_
3. Similarly, run kubectl to check keytab is deployed as a secret, keytab should be listed as a K8 secret,
_kubectl_.exe get secrets -n=kube-system

Few things to note before proceeding to the next steps

1. Once the AD webhook and keytab are successfully deployed, generate the AD kubeconfig
2. The AD kubeconfig should be copied over to the client machine and this is used to authenticate to the api-server, unlike certificate based kubeconfig, AD kubeconfig is not a secret and is safe to copy as plain text

_Step 4: Generate the AD kubeconfig_

_Get-AksHciCredential -clusterName mynewcluster1 -outputLocation .\AdKubeconfig -adAuth_

_Step 5: Copy the following files from the server to your client machine, create a folder c:\adsso and copy the files_

1. AdKubeconfig file created in the previous step
2. Kubectl.exe under $env:ProgramFiles\AksHci
3. Kubectl-adssso.exe under $env:ProgramFiles\AksHci

# **Part B: Creating and Updating AD group role binding**

As mentioned in step 2, a default role binding is created for the mentioned group with cluster admin privileges.

While creating or editing additional AD group RBAC entries, the subject name should be pre-fixed by &quot;microsoft:activedirectory:CONTOSO\\&lt;group name\&gt;&quot;

Note: Before deploying the yaml the \&lt;group name\&gt; **should always be converted to SID using the command**

_Kubectl-adsso.exe nametosid \&lt;rbac.yml\&gt;_

Similarly, in order to update an existing rbac, SID can be converted to user friendly group name before making changes

_Kubectl-adsso.exe sidtoname \&lt;rbac.yml\&gt;_

# **Part C: Troubleshooting and Maintenance**

## Changing the AD account password associated with api-server account


When the password is changed for the api-server account, uninstall the AD Auth add-on and reinstall it using the updated current and previous keytabs. **The files must be named current.keytab and previous.keytab respectively**. Note, the existing role bindings are not affected by this.

_Uninstall-AksHciAdAuth -clusterName \&lt;mynewcluster\&gt;_

_Install-AksHciAdAuth -clusterName \&lt;mynewcluster\&gt; -keytab \&lt;.\current.keytab\&gt; -previousKeytab \&lt;.\previous.keytab\&gt; -SPN \&lt;service/principal@CONTOSO.COM\&gt; -adminUser \&lt;CONTOSO\Bob\&gt;_

**The keytabs should be named exactly as current.keytab and previous.keytab**

## Uninstall and reinstall AD SSO

Use the following command to uninstall and re-install AD SSO, note, this doesn&#39;t require the api-server to be restarted

_Uninstall-AksHciAdAuth -clusterName \&lt;mynewcluster\&gt;_

_Install-AksHciAdAuth -clusterName \&lt;mynewcluster\&gt; -keytab \&lt;.\current.keytab\&gt; -SPN \&lt;service/principal@CONTOSO.COM\&gt; -adminUser \&lt;CONTOSO\Bob\&gt;_

# **Additional: Creating api-server AD Account, SPN and keytab**

There are two steps involved in this process. First, create a new AD account/user for the api-server with service principal option. Next, create keytab file for the AD account.

_Step 1: Create a new AD account / user for the api-server with service principal option_

_New-ADUser -Name &quot;\&lt;apiserver\&gt;&quot;_

_-ServicePrincipalNames &quot;\&lt;K8s/apiserver\&gt;&quot;_

_-AccountPassword (ConvertTo-SecureString &quot;\&lt;password\&gt;&quot; -AsPlainText -Force)_

_-KerberosEncryptionType \&lt;AES128,AES256\&gt;_

_-Enabled 1_

_Example:_

_New-ADUser -Name &quot;apiserver\_acct&quot; -ServicePrincipalNames &quot;k8s/apiserver&quot; -AccountPassword (ConvertTo-SecureString &quot;p@$$w0rd&quot; -AsPlainText -Force) -KerberosEncryptionType AES256 -Enabled 1_

_Step 1: Create a new AD account / user for the api-server with service principal option_

_ktpass /out \&lt;current.keytab\&gt; /princ \&lt;service/principal@CONTOSO.COM\&gt; /mapuser \&lt;Domain\account\&gt; /crypto all /pass \&lt;password\&gt; /ptype \&lt;KRB5\_NT\_PRINCIPAL\&gt;_

_Example:_

_ktpass /out current.keytab /princ k8s/apiserver@BAKSDOM.NTTEST.MICROSOFT.COM /mapuser baksdom\apiserver\_acct /crypto all /pass p@$$w0rd /ptype KRB5\_NT\_PRINCIPAL_

# **Reference: How to get SID**

## Getting SID associated with your account

On the command line on your HOME directory type _whoami/user_

## Getting SID associated with another account

On the command line on your HOME directory

- Type wmic
- Wmic prompt will show up
- Then type useraccount where name=&quot;\&lt;USERNAME\&gt;&quot; get sid