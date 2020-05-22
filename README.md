# Create a ZIP package for Powershell extensibility in ABX ( and vRO ) actions
# including proprietary Powershell Modules and Proxy Options.

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

These instructions will fetch the proxy settings from 'context' object of the function handler and set it as enviroment property. It is important to mention that ABX extensibility uses the same vRA Proxy settings 

You can verify your vRA Proxy settings with the command *vracli proxy 

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
 *https://hub.docker.com/r/vmware/powerclicore/
 *More details here: https://github.com/vmware/powerclicore

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


# Create and activate a new Python environment:

Create an activate a python3 development environment 
( Please note my $HOME "/root" may be different than yours please adapt the folder locations )

	root@ubuntu_server: mkdir environments
	root@ubuntu_server: python3 -m vraDNSDev 
	
Create and move to the root folder for your ABX Action

	(vraDNSDev) root@ubuntu_server: mkdir vraDNS-action    
	(vraDNSDev) root@ubuntu_server: cd /root/enviroments/vraDNSDev/vraDNS-action

# Define your library requirements and install them with PIP at your action root folder

Copy or Create then place the requirements.txt inside your ABX action root folder 
For our example only the "dnspython==1.16.0" propetary library is needed, which it is not included in the standard Python or FaaS Engines

	(vraDNSDev) root@ubuntu_server: vi requirements.txt 
		dnspython==1.16.0     
		
Now install "dnspython==1.16.0" in your Python virtual enviroment

	(vraDNSDev) root@ubuntu_server:  pip install -r requirements.txt --target=/root/enviroments/vraDNSDev/vraDNS-action   

You should see the following folders:

	(vraDNSDev) root@ubuntu_server:~/enviroments/vraDNSDev/vraDNS-action# ls -lrt
	total 408
	drwxr-xr-x 4 root root   4096 Mar  3 01:00 dns
	drwxr-xr-x 2 root root   4096 Mar  3 01:00 dnspython-1.16.0.dist-info
	-rw-r--r-- 1 root root    759 Mar  3 17:21 main.py
	-rw-r--r-- 1 root root     18 Mar  3 17:46 requirements.txt

My principal and only Python Script is "main.py"
It is a basic sample for translating a MSISDN number into ENUM format calling dnspython's dns.e164.from_e164()
it also resolves and lists the MX records for a given domain via dnspython's dns.resolver.query()
and finally resolve A record with prebuilt python's socket library

/////////main.py/////////////////

	import socket
	import dns.resolver
	import dns.e164

	def handler(context, inputs):
	    print('Action started.')
	    msAddr = inputs["msisdn"]
	    dnsMX = inputs["dnsMX"]

	    # Log Input Entries
	    #print (inputs)
	    #print (msAddr)
	    #print (dnsMX)

	    # Converts E164 Address to ENUM with propietary library dnspython
	    n = dns.e164.from_e164(msAddr)
	    print ('My MSISDN:', dns.e164.to_e164(n), 'ENUM NAME Conversion:', n)

	    # Resolve DNS MX Query with propietary library dnspython
	    answers = dns.resolver.query(dnsMX, 'MX')
	    print ('Resolving MX Records for:', dnsMX)
	    for rdata in answers:
		print ('Host', rdata.exchange, 'has preference', rdata.preference)

	    # Resolve AAA with prebuilt socket library
	    addr1 = socket.gethostbyname(dnsMX)
	    print('Resolving AAA Record:', addr1)

	    return addr1

Now let's package the main Script with the customized installed libraries
Both your script and dependency elements must be stored at the root level of the ZIP package. When creating the ZIP package in a Linux environment, you might encounter a problem where the package content is not stored at the root level. If you encounter this problem, create the package by running the zip -r command in your command-line shell.

	(vraDNSDev) root@ubuntu_server: cd ~/enviroments/vraDNSDev/vraDNS-action
	(vraDNSDev) root@ubuntu_server: zip -r9 vraDNS-actionR08.zip *

Let's use now the ZIP package to create an extensibility action script by importing it at vRA
Log In to vRA with a user having Cloud Assembly Permissions
Go to [ Cloud Assembly ]--> [ Extensibility ] --> [ Actions ] --> [ Create a New Action ] and associate to your Project

   ![New Action](https://github.com/moffzilla/vraDNS-action/blob/master/media/newAction.png) 

Instead of "Write Script", Select Import Package and import your zip file (e.g. vraDNS-actionR08.zip is a pre-staged working action) 

   ![importAction](https://github.com/moffzilla/vraDNS-action/blob/master/media/importAction.png) 

Define inputs required by the script ( see defaults below ) and define the Main Function as point of entry 

   ![inputAction](https://github.com/moffzilla/vraDNS-action/blob/master/media/inputAction.png) 

Please note that for actions imported from a ZIP package, the main function must also include the name of the script file that contains the entry point. For example, if your main script file is titled main.py and your entry point is handler (context, inputs), the name of the main function must be main.handler.

You can select your prefered FaaS Provider or simply let vRA to do it for you by selecting "Auto"

Save and Test your ABX Action

 ![saveAction](https://github.com/moffzilla/vraDNS-action/blob/master/media/saveAction.png) 
 
 Click on "See Details" to see your Python Script execution details
 Please note that the first time you execute it, it takes more time as it needs to upload your action to your local or remote FaaS providers

 ![detailsAction](https://github.com/moffzilla/vraDNS-action/blob/master/media/detailsAction.png)
 
You can change the input and FaaS provider ( please note that the MSISDN is in international format and dnsMX don't need WWW as it is an EMAIL Exchange Record)

From this point you could create an Extensibility Subscrition or expose this action at vRA's Service Broker Catalog

Don't forget to deactivate your Python enviroment

	(vraDNSDev) root@ubuntu_server:~/enviroments# deactivate
	root@ubuntu_server:~/enviroments#
