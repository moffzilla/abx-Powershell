# Create a ZIP package including proprietary Powershell Modules for vRA ABX

There are a few methods of building the script for your extensibility actions
Writing your Powershell script directly in the extensibility action editor in vRealize Automation Cloud Assembly.
Please note that depending on the Powershell Modules, you could instruct to load them directly at the Action Dependencies
e.g. 

  ![az-latest](https://github.com/moffzilla/abx-Powershell/blob/master/media/az-latest.png)
  
  
Also note that this option is valid if you hace connectivity to the public repositories where the modules are stored.

In situations where you don't have access to those public repositories you need to pre-install those modules 
and you do this by simply creating your main script on your local Linux based Powershell environment, pre-install the required Modules for your script then bundle all together into ZIP package that later you can import as a whole to vRA Cloud Assembly for use in extensibility actions.

Additionally, in some occasions, you may need to call a public service like MS Azure or any other that is only reachable behind a Proxy and depending of the nature of your Powershell Scripts or selected modules, e.g. some modules read their proxy settings from OS environment variables, in those situations you have to implicitly specify your proxy settings

	$proxyString = "http://" + $context.proxy.host + ":" + $context.proxy.port
	$Env:HTTP_PROXY = $proxyString
	$Env:HTTPS_PROXY = $proxyString 

These instructions will fetch the proxy settings from 'context' object of the function handler and set it as environment property. It is important to mention that ABX extensibility uses the same vRA Proxy settings 

You can verify your vRA Proxy settings with the command *vracli proxy you have to implicitly specify your proxy settings

	root@cava-n-81-246 [ ~ ]# vracli proxy show
	{
	    "enabled": true,
	    "host": "10.244.4.51",
	    "java-proxy-exclude": "*.local|*.localdomain|localhost|10.244.*|192.168.*|172.16.*|kubernetes|cava-n-81-246.eng.vmware.com|10.149.81.246|*.eng.vmware.com",
	    "port": 3128,
	    "proxy-exclude": ".local,.localdomain,localhost,10.244.,192.168.,172.16.,kubernetes,cava-n-81-246.eng.vmware.com,10.149.81.246,.eng.vmware.com",
	    "scheme": "http",
	    "upstream_proxy_password_encoded": "",
	    "upstream_proxy_user_encoded": "",
	    "internal.proxy.config": "dns_v4_first on \nhttp_port 0.0.0.0:3128\nlogformat squid %ts.%03tu %6tr %>a %Ss/%03>Hs %<st %rm %ru %[un %Sh/%<a %mt\naccess_log stdio:/tmp/logger squid\ncoredump_dir /\ncache deny all \nappend_domain .prelude.svc.cluster.local\nacl mylan src 10.0.0.0/8\nacl mylan src 127.0.0.0/8\nacl mylan src 192.168.3.0/24\nacl proxy-exclude dstdomain .local\nacl proxy-exclude dstdomain .localdomain\nacl proxy-exclude dstdomain localhost\nacl proxy-exclude dstdomain 10.244.\nacl proxy-exclude dstdomain 192.168.\nacl proxy-exclude dstdomain 172.16.\nacl proxy-exclude dstdomain kubernetes\nacl proxy-exclude dstdomain 10.149.81.246\nacl proxy-exclude dstdomain .eng.vmware.com\nalways_direct allow proxy-exclude\nhttp_access allow mylan\nhttp_access deny all\n# End autogen configuration\n",
	    "internal.proxy.config.type": "default"
	}
	root@cava-n-81-246 [ ~ ]#

On top of that, using extension ".psm1" ( which is standard for dependencies modules for pwsh engine requirements ) is recommended instead the common ".ps1" for the handler (entrypoint) function file.
This is relevant as all additional dependencies packaged will be automatically loaded, meaning that all the "imports" statements at beginning of the file are not needed and need to be removed. 

Any pre-installed modules must be placed at root level in order to be automatically imported ( more in the example below )

# Requirements
    vRA 8.X ( tested on 8.1 ) or vRA Cloud with ABX on-prem
    Ubuntu 18.04.4 LTS 
    PowerShell 6.2.3 is the recommended version, however I am able to stage and install Modules with PowerShell 7.0.0 for this example
    
I would recommend to use the same Powershell version shiped with vRA 8.1 which can be found here 

https://hub.docker.com/r/vmware/powerclicore/

 More details here: https://github.com/vmware/powerclicore

Please note that the runtime of action-based extensibility in vRealize Automation Cloud Assembly is Linux-based.
Therefore, any Powershell dependencies compiled in a Windows environment might make the generated ZIP package unusable for the creation of extensibility actions, always use a Linux shell ( Photon OS preferable but Ubuntu 18.04 would work ).


# Install Powershell

Ubuntu 18.04 you can install Powershell for Linux as follows:

	sudo apt update
	sudo apt -y upgrade
	sudo apt-get install -y powershell
	
check the version of  by typing:

	pwsh --version

Output

       PowerShell 7.0.0


# Pre-install your Powershell Modules in your action root folder

In your local staging system create your ABX action root folder 

	mkdir abx-powershell
	cd abx-powershell
	mkdir Modules
	
Install your modules from PSGallery Repository https://www.powershellgallery.com/

	pwsh -c Get-PSRepository
	
	Name                      InstallationPolicy   SourceLocation
	----                      ------------------   --------------
	PSGallery                 Untrusted            https://www.powershellgallery.com/api/v2

In this example, the "Az" Module will be used https://www.powershellgallery.com/packages/Az/4.1.0

	pwsh -c "Save-Module -Name Az -Path ./Modules/ -Repository PSGallery"  
		
This will import all the default modules, you can manually remove the ones you don't need and/or install specifically  the ones your script requires ( it can be loaded into execution faster )

	root@ubuntu_server:~/abx-powershell/Modules#  ls -lrt
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.Maintenance
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.Media
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.Monitor
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.OperationalInsights
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.PrivateDns
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.RecoveryServices
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.Resources
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.ServiceBus
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.ServiceFabric
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.Accounts
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.Accounts
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.Advisor
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.Aks
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.AnalysisServices
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.ApiManagement
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.ApplicationInsights
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.Automation
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.Batch
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.Billing
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.Cdn
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.CognitiveServices
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.Compute
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.ContainerInstance
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.ContainerRegistry
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.DataBoxEdge
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.DataFactory
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.DataLakeAnalytics
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.DataLakeStore
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.DeploymentManager
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.DevTestLabs
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.Dns
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.EventGrid
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.EventHub
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.FrontDoor
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.Functions
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.HDInsight
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.HealthcareApis
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.IotHub
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.KeyVault
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.LogicApp
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.MachineLearning
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.Maintenance
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.ManagedServices
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.MarketplaceOrdering
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.Media
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.Monitor
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.Network
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.NotificationHubs
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.OperationalInsights
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.PolicyInsights
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.PowerBIEmbedded
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.PrivateDns
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.RecoveryServices
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.RedisCache
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.Relay
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.Resources
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.ServiceBus
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.ServiceFabric
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.SignalR
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.Sql
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.SqlVirtualMachine
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.Storage
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.StorageSync
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.StreamAnalytics
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.Support
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.TrafficManager
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az.Websites
	drwxr-xr-x 3 root root 4096 May 22 02:50 Az
	root@ubuntu_server:~/abx-powershell/Modules# 


My principal and only Powershell Script is 
	
	handler.psm1
	
Here's an extract of the whole script, please note the Proxy related instructions since vRA is behind a proxy and Powershell needs to load the proxy settings from environment
Also there is no need to indicate to load any modules, this will be done automatically.

/////////handler.psm1/////////////////

	function handler {
	  Param($context, $inputs)
	  $inputsString = $inputs | ConvertTo-Json -Compress
	  $DebugPreference = "Continue"

	  $proxyString = "http://" + $context.proxy.host + ":" + $context.proxy.port
	  $Env:HTTP_PROXY = $proxyString
	  $Env:HTTPS_PROXY = $proxyString

	  Write-Host $proxyString
	  $proxyUri = new-object System.Uri($proxyString)
	  [System.Net.WebRequest]::DefaultWebProxy = new-object System.Net.WebProxy ($proxyUri)
	  Write-Host "Inputs were $inputsString"
	  # From this point it is my script making calls to MS Azure
	  .......

Now let's package the main handler.psm1 script with the customized installed Modules
Both your script and dependency elements must be stored at the root level of the ZIP package. 
When creating the ZIP package in a Linux environment, you might encounter a problem where the package content is not stored at the root level, make sure to create the package by running the zip -r command in your command-line shell.

	root@ubuntu_server:~/powershell/abx-powershell# zip -r --exclude=*.zip -X VRA_Powershell_vro_05.zip .
	 
At this point we can use the ZIP package to create an extensibility action script by importing it at vRA
Log In to vRA with a user having Cloud Assembly Permissions
Go to [ Cloud Assembly ]--> [ Extensibility ] --> [ Actions ] --> [ Create a New Action ] and associate to your Project

   ![New Action](https://github.com/moffzilla/abx-Powershell/blob/master/media/newAction.png) 

Select "Powershell" and Instead of "Write Script", Select Import Package and import your zip file 
	
	(e.g. VRA_Powershell_vro_05.zip is a pre-staged working action) 

   ![importAction](https://github.com/moffzilla/abx-Powershell/blob/master/media/importAction.png) 

Define inputs required by the script ( see defaults below ) and define the Main Function as point of entry 

	handler.handler

   ![inputAction](https://github.com/moffzilla/abx-Powershell/blob/master/media/inputAction.png) 

Please note that for actions imported from a ZIP package, the main function must also include the name of the script file that contains the entry point. 

You can select your prefered FaaS Provider or simply let vRA to do it for you by selecting "Auto"

Save and Test your ABX Action
Click on "See Details" to see your Powershell Script execution details
Please note that the first time you execute it, it takes more time as it needs to upload your action to your local or remote FaaS providers, subsequent executions will be faster.

 ![detailsAction](https://github.com/moffzilla/abx-Powershell/blob/master/media/detailsAction.png)
 
You can change the input and FaaS provider as needed.


