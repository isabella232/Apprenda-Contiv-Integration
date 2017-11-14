# Contiv Networking Integration
This integration provides automatic application-level network configuration using Contiv and ACI for Kubernetes workloads deployed with Apprenda.

### Supported Software Versions
|Software|Min Version|Max Version|
|-|-|-|
|Apprenda|7.0|8.0|
|Kubernetes|1.6.4|1.7.5|
|Contiv|1.0.3|1.1.6|
|ACI|2.3|3.0|

### Informational Videos
[Integrated Container Networking Overview](https://youtu.be/1qQ7lj6fIt4)

### Installation Videos
[Installation Part 1: ACI Setup](https://youtu.be/QfpvOlhUcFQ)

[Installation Part 2: Contiv Setup](https://youtu.be/hypkXX7zeLM)

[Installation Part 3: Installation](https://youtu.be/MuVIuJDMgQ4)

[Installation Part 4: Contiv Setup](https://youtu.be/-c77MNnPp8I)

## Prerequisites
If using ACI, this guide assumes a functional Apprenda instance running on ESXi hosts. It also assumes that ACI has been configured for use with the ESXi hosts.

### ACI Reference Architecture
![ACI Reference Architecture](/docs/aci_reference_architecture.png)

This integration relies on existing Apprenda and L3 Out contracts in the common tenant for communication with Apprenda nodes and resources external to ACI. The default tenant does not need to be created beforehand as it will be created by the integration's CLI.

### Kubernetes Installation with Contiv
*We'll be using the [Kismatic Enterprise Toolkit (KET)](https://github.com/apprenda/kismatic) for ease of installing Kubernetes*
1. Follow steps provided in the [guide](https://github.com/apprenda/kismatic/blob/master/docs/install.md) in order to generate your kismatic-cluster.yaml
2. Specify your CNI
    * For Contiv with ACI, you must specify that you don't want KET to manage your CNI. Modify your kismatic-cluster.yaml and change the provider to "custom" as shown below:
      ```
      add_ons:
        cni:
          disable: false
          provider: custom
      ```
    * For Contiv without ACI, in order to specify that you want to use Contiv as your CNI (instead of Calico/Weave), modify your kismatic-cluster.yaml and change the provider to "contiv" as shown below:
	    ```
      add_ons:
        cni:
          disable: false
          provider: contiv
      ```
3. Install Kubernetes using the following command:
   ```
   kismatic install apply
   ```
4. For Contiv with ACI, you must install Contiv manually using the steps below

### Contiv Installation
Contiv with ACI requires an additional NIC. We recommend configuring an additional distributed port group in VMware for this purpose. The port group should be configured for the same VLAN range as the Default-PD physical domain in ACI and also must have forged transmits enabled to accept traffic from Docker containers.

1. Use the following command to install Contiv, replacing the environment specific arguments where applicable:
	```
	./install/k8s/install.sh -n <master node IP> -a <APIC url> -u username -p password -l <leaf> -d <physical domain> -e not_specified -m no -w bridge -v <additional interface>
	```
2. Use the following command to set the fabric mode and VLAN range, replacing the environment specific arguments where applicable:
	```
	netctl global set --fabric-mode aci -b bridge -a flood --vlan-range <physical domain VLAN range>
	```

## Initialization
*Subnet ranges must match when initializing ACI and Contiv*
1. If using ACI, initialize ACI by running the following command from the LM:
   ```
   ContivProxy.CLI.exe InitializeAci ...
   ```
   * Use the ```-V``` and ```-L``` arguments to specify the name of the Apprenda VRF and L3 Out in your ACI setup
2. Initialize Contiv by running the following command from the LM:
   ```
   ContivProxy.CLI.exe InitializeContiv ...
   ```
   * If using ACI, use the ```-C``` argument to specify the Apprenda and L3 Out contracts in your ACI setup
   * When complete, use kubectl to scale kube-dns to zero and back to one (this will ensure its deployment to default-group)

## Installation
1. Create a domain account for the integration
2. Run GenerateCertificate.ps1 to generate the credentials-encrypting certificate
3. Copy ContivProxy.pfx (created by step 2) and ProvisionNode.ps1 to each Windows node
4. Run ProvisionNode.ps1 on each Windows node
    * This will import the ContivProxy certificate, grant the domain account read access to the private key, and grant "Log on as service" to the domain account
5. Run CustomizeArchive.ps1 to embed the domain credentials in the archive
6. Copy ContivProxy.zip, ContivProxy.Bootstrapper.zip, and ConfigureApprenda.ps1 to the LM
7. Run ConfigureApprenda.ps1 on the LM (Apprenda SDK required)
    * This will create and promote the Contiv Proxy application, create all the necessary Custom Properties, and create the Contiv Proxy Bootstrapper (it will also set credentials if specified)
    
## Verification
1. Verify that credentials are set by launching the Contiv Proxy UI
2. Create a new application using the provided connectivity-pod.yaml
3. Set the component custom property "Apprenda Network" to "Development Team"
4. Add "0/ICMP" and "3000/TCP" to component custom property "Application Ports"
5. Promote the application to sandbox
    * Use APIC to verify that an EPG with appropriate contracts was created
    * Verify that Connectivity Pod web UI can be reached by its Apprenda URL
    * Use the Connectivity Pod web UI to verify connectivity with outside resources
6. Demote the application
    * User APIC to verify that the EPG was removed

## Troubleshooting
The log can be accessed by launching the application and navigating to /log.
* Failure to initialize
  1. Check for connectivity between the LM and Contiv/ACI
  2. Check that correct service URLs and credentials are being passed to the CLI
* Failure to save credentials
  1. Check that all Windows nodes have the "ContivProxy" certificate installed under LocalMachine/Personal/Certificates
* Failure to promote
  1. Check that correct service URLs and credentials are configured
  2. Check that all Windows nodes have the ContivProxy certificate installed under LocalMachine/Personal/Certificates and that the domain account has been granted read access to the private key
  3. Check that Contiv has been correctly configured for the ACI environment
* Failure to remove EPG
  1. Check that the workload had been removed from Kubernetes
  2. Check that all Windows nodes have the ContivProxy certificate installed under LocalMachine/Personal/Certificates and that the domain account has been granted read access to the private key
  3. Check that the domain account has been granted "Log on as service" in Local Group Policy on all Windows nodes
