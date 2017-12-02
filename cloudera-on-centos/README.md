# Readme
This template creates a multi-server Cloudera CDH Apache Hadoop deployment on CentOS virtual machines, and configures the CDH installation for either POC or high availability production cluster.

The template also provisions storage accounts, virtual network, availability set, network interfaces, VMs, disks and other infrastructure and runtime resources required by the installation.

The template expects the following parameters:

| Name   | Description | Default Value |
|:--- |:---|:---|
| adminUsername  | Administrator user name used when provisioning virtual machines | azureuser |
| adminPassword  | Administrator password used when provisioning virtual machines | {No Default} |
| cmUsername | Cloudera Manager username | cmadmin |
| cmPassword | Cloudera Manager password | {No Default} |
| storageAccountSuffix | Unique namespace for the Storage Account where the Virtual Machine's disks will be placed | {No Default} |
| numberOfDataNodes | Number of data nodes to provision in the cluster | 3 |
| dnsNamePrefix | Unique public dns name where the Virtual Machines will be exposed | {No Default} |
| region | Azure data center location where resources will be provisioned | {No Default} |
| masterStorageAccountType | The type of the Storage Account to be created for master nodes | Premium_LRS |
| workerStorageAccountType | The type of the Storage Account to be created for worker nodes | Standard_LRS |
| virtualNetworkName | The name of the virtual network provisioned for the deployment | clouderaVnet |
| subnetName | Subnet name for the virtual network where resources will be provisioned | clouderaSubnet |
| tshirtSize | T-shirt size of the Cloudera cluster (Eval, Prod) | Eval |
| vmSize | The size of the VMs deployed in the cluster (Defaults to Standard_DS14) | Standard_DS14 |
| vmImage | The OS VM Image (defaults to ClouderaCentOS6_7) | ClouderaCentOS6_7 |


Topology
--------

The deployment topology is comprised of a predefined number (as per t-shirt sizing) Cloudera member nodes configured as a cluster, configured using a set number of manager,
name and data nodes. Typical setup for Cloudera uses 3 master nodes with as many data nodes are needed for the size that has been choosen ranging from as
few as 3 to thousands of data nodes.  The current template will scale at the highest end to 200 data nodes when using the large t-shirt size.

The following table outlines the deployment topology characteristics for each supported t-shirt size:

| T-Shirt Size | Member Node VM Size | CPU Cores | Memory | Data Disks | # of Master Node VMs | Services Placement of Master Node |
|:--- |:---|:---|:---|:---|:---|:---|:---|
| Eval | Standard_DS14 | 10 | 112 GB | 10x1000 GB | 1 | 1 (primary, secondary, cloudera manager) |
| Prod | Standard_DS14 | 10 | 112 GB | 10x1000 GB | 3 | 1 primary, 1 standby (HA), 1 cloudera manager |

##Connecting to the cluster
The machines are named according to a specific pattern.  The master node is named based on parameters and using the.

	[dnsNamePrefix]-mn0.[region].cloudapp.azure.com

If the dnsNamePrefix was clouderatest in the West US region, the machine will be located at:

	clouderatest-mn0.westus.cloudapp.azure.com

The rest of the master nodes and data nodes of the cluster use the same pattern, with -mn and -dn extensions followed by their number.  For example:

    clouderatest-mn0.westus.cloudapp.azure.com
	clouderatest-mn1.westus.cloudapp.azure.com
	clouderatest-mn2.westus.cloudapp.azure.com
	clouderatest-dn0.westus.cloudapp.azure.com
	clouderatest-dn1.westus.cloudapp.azure.com
	clouderatest-dn2.westus.cloudapp.azure.com

To connect to the master node via SSH, use the username and password used for deployment

	ssh testuser@[dnsNamePrefix]-mn0.[region].cloudapp.azure.com

Once the deployment is complete, you can navigate to the Cloudera portal to watch the operation and track it's status. Be aware that the portal dashboard will report alerts since the services are still being installed.

	http://[dnsNamePrefix]-mn0.[region].cloudapp.azure.com:7180

## Notes, Known Issues & Limitations
- All nodes in the cluster have a public IP address.
- The deployment script is not yet idempotent and cannot handle updates (although it currently works for initial provisioning only)
- SSH key is not yet implemented and the template currently takes a password for the admin user
- Deployment logs can be found at `/var/log/cloudera-azure-initialize.log`.

## Important Notes on CLI Deployment
If you attempt to deploy a local template file via Azure PowerShell or the Azure CLI, the *scriptsUri* variable cannot be constructed automatically and must be edited point to an Azure accessible location with the remainder of the templates and scripts.  If you are only changing the top level template (azuredeploy.json, azuredeployds13.json or azuredeployGermany.json), then you can use the example URI in *scriptsUriDescription* to point *scriptsUri* to the remaining files in the main template publishing location.

## How to Access Cloudera Manager and other Cloudera services deployed by this template

By default the Network Security Groups in this offering only allow public IP access to port 22 (SSH).

There are two ways to access services like Cloudera Manager (port 7180):
1. Use a SOCKS proxy (preferred)
1. Add inbound rules to the Network Security Group to allow access to ports.

The two solutions are detailed below.

### SOCKS Proxy

A SOCKS proxy changes your browser to do lookups directly from your Microsoft Azure network and allows you to connect to services using private IP addresses and internal FQDNs.

This approach will do the following:
* Set up a single SSH tunnel to one of the hosts on the network and create a SOCKS proxy on that host.
* Change the browser configuration to do all lookups via that SOCKS proxy host.

#### Network Prerequisites

The following are prerequisites for connecting to your cluster using a SOCKS proxy:
* The host that you proxy to must be reachable from the public internet (or the network that you’re connecting from).
* The host that you proxy to must be on the same network as the Cloudera services you’re connecting to. For example, with this Cloudera EDH offering, tunnel to the Cloudera Manager host.

#### Find the Public IP of the Cloudera Manager host

Use the public IP of the 0th master node VM. In the Azure Portal the VM will have a name that looks like:
```
[dnsNamePrefix]-mn0
```

#### Start the SOCKS Proxy

##### On Linux

To start a SOCKS5 proxy over SSH run the following command and input the VM password at the prompt:
```
ssh -CND 1080 username@publicIP_of_VM
```
Or, if you’re using SSH keys:
```
ssh -i your-key-file.pem -CND 1080 username@publicIP_of_VM
```
The parameters are as follows:

* `i your-key-file.pem` specifies the path to the private key
* `C` sets up compression
* `N` suppresses any command execution once established
* `D` sets up the SOCKS proxy on a port
* `1080` is the port to set the SOCKS proxy locally

##### On Windows
[Follow the instructions here](https://blogs.msdn.microsoft.com/pliu/2017/01/17/ssh-tunnel-to-endpoints-in-azure-vnet-from-windows/).

#### Configure Your Browser to Use the Proxy

##### On Google Chrome

By default, Google Chrome uses system-wide proxy settings on a per-profile basis. To get around that we will launch Chrome via the command line and specify the following:
* The SOCKS proxy port to use (this must be the same value used above)
* The profile to use (this example will create a new profile)

This will create a new profile and launch a new instance of Chrome that won’t interfere with your current running instance of Chrome.

**Linux**
```
/usr/bin/google-chrome \
--user-data-dir="$HOME/chrome-with-proxy" \
--proxy-server="socks5://localhost:1080"
```

**Mac OS X**
```
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
--user-data-dir="$HOME/chrome-with-proxy" \
--proxy-server="socks5://localhost:1080"
```

**Microsoft Windows**
```
"C:\Program Files (x86)\Google\Chrome\Application\chrome.exe" ^
--user-data-dir="%USERPROFILE%\chrome-with-proxy" ^
--proxy-server="socks5://localhost:1080"
```

Now in this Chrome session you can connect to any other host on the Virtual Network using the private IP address or internal FQDN. For example, if you proxy to Cloudera EDH Master Node 0 you can connect to Cloudera Manager at `localhost:7180`.


### Network Security Group

Warning: this method is **NOT** recommended for anything besides a PoC. If not carefully locked down the data in the cluster will be accessible to hackers and malicious entities.

On [portal.azure.com](https://portal.azure.com) find the Network Security Group(s) and add inbound rules for the various services. You may have to create this rules in multiple Network Security Groups. [Here's Cloudera documentation for more information on ports used by Cloudera Manager, CDH components, managed services, and third-party components](http://www.cloudera.com/documentation/enterprise/latest/topics/cm_ig_ports.html).
