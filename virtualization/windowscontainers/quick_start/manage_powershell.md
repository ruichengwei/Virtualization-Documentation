ms.ContentId: d0a07897-5fd2-41a5-856d-dc8b499c6783
title: Manage Windows Server Containers with PowerShell

#Manage Windows Server Containers with PowerShell

This article will demonstrate basic use of PowerShell with Windows Server Containers, basic image and container management, and a simple yet practical use for Windows Server Containers. The lessons learned in this walkthrough should enable you to begin exploring deployment and management of Windows Server Containers using PowerShell.

If you’ve used Hyper-V PowerShell, the design of the cmdlets for Windows Server Containers should be pretty familiar to you. A lot of the workflow is similar to how you’d manage a virtual machine using the Hyper-V module. Instead of `New-VM`, `Get-VM`, `Start-VM`, `Stop-VM`, you have `New-Container`, `Get-Container`, `Start-Container`, `Stop-Container`. There are quite a few container-specific cmdlets and parameters, but the general lifecycle and management of a Windows containers with PowerShell looks roughly like that of a Hyper-V VM.

> PLease Note - Windows Containers created with Docker need to be managed with Docker. For more information, see [Managing Windows Containers with Docker](./manage_docker.md).

##Basic Container Management with PowerShell:

This first example will walk through the basics of managing container with PowerShell such as creating and removing containers and container images.

##Step 1 - Create a Container
Start a PowerShell session from the command prompt by typing `PowerShell`. You will know that you are in a PowerShell session when the prompt changes from `C:\directory>` to `PS C:\directory>`.

Before creating a Windows Server Container you will need the name of a Container Image and the name of a virtual switch that will be attached to the container.

Use the `Get-ContainerImages` command to return a list of images loaded on the host. Take note of the image name that you will use to create a container.

```powershell
Get-ContainerImage

Name              Publisher    Version      IsOSImage
----              ---------    -------      ---------
WindowsServerCore CN=Microsoft 10.0.10254.0 True
```

Use the `Get-VMSwitch` command to return a list of switches available on the host. Take note of the switch name that will be attached to the container.

```powershell
Get-VMSwitch

Name           SwitchType NetAdapterInterfaceDescription
----           ---------- ------------------------------
Virtual Switch External   Microsoft Hyper-V Network Adapter
```

Run the following command to create a container with a name of 'MyContainer'. Notice that this example places the output of `New-Container` into a variable `$container`. This will be helpful later on.

```powershell
$container = New-Container -Name "MyContainer" -ContainerImageName WindowsServerCore -SwitchName "Virtual Switch"
```

To see a list of containers on the host and verify that the container was created, use the `Get-Container` command.

```powershell
Get-Container

Name        State Uptime   ParentImageName
----        ----- ------   ---------------
MyContainer Off   00:00:00 WindowsServerCore
```

To start the container use `Start-Container`.

```
Start-Container -Name "MyContainer"
```
When a container has been created you can interact with it using PowerShell remoting commands such as `Invoke-Command`, or `Enter-PSSession`. The example below creates a remote PowerShell session into the container using the `Enter-PSSession` command. Notice that once the remote session has been created the command prompt will change to include the first 11 characters of the container id `[2446380e-629]`.

```powershell
Enter-PSSession -ContainerId $container.ContainerId -RunAsAdministrator

[2446380e-629]: PS C:\Windows\system32>
```

A container can be managed very much like a physical or virtual machine. Command such as `ipconfig` to return the IP address of the container, `mkdir` to create a directory in the container and PowerShell commands like `Get-ChildItem` all work. Go ahead and make a change to the container such as creating a file or folder. For example, the following command will create a file which contains network configuration data about the container.

```
ipconfig > c:\ipconfig.txt
```

You can read the contents of the file to ensure the command completed successfully. Notice that the IP address contained in the text file matches that of the container.
```
type c:\ipconfig.txt

Ethernet adapter vEthernet (Virtual Switch-b34f32fcdc63b8632eaeb114c6eb901f8982bc91f38a8b64e6da0de40ec47a07-0):

   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::85b:7834:454c:375b%20
   IPv4 Address. . . . . . . . . . . : 192.168.1.55
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . :

```
Now that the container has been modified, exit the remote PSSession by typing `exit` and stop the container using the `Stop-Container` command. When these commands have completed, you will be back in control of the container host.

```powershell
#Exit the remote PowerShell session.
exit

#Stop the container.
Stop-Container -Container $container
```

##Step 2 - Create a Container Image

An image can now be made from this container. This image will behave like a snapshot of the container and can be re-deployed many times.

To create an image with the name of ‘newimage’ run the following: 

```powershell
$newimage = New-ContainerImage -Container $container -Publisher Demo -Name newimage -Version 1.0
```

Use `Get-ContainerImage` to return a list of Container Images. Notice that a new image has been created.

```powershell
Get-ContainerImage

Name              Publisher    Version      IsOSImage
----              ---------    -------      ---------
newimage          CN=Demo      1.0.0.0      False
WindowsServerCore CN=Microsoft 10.0.10254.0 True
```

## Step 3 - Create Container From Image

Create a container named 'newcontainer' from the new container image.
```powershell
$newcontainer = New-Container -Name "newcontainer" -ContainerImageName newimage -SwitchName "Virtual Switch"
```
Start the container and create a PSSession into the new container.
```powershell
#Start Container
Start-Container $newcontainer

#Create PowerShell connection into container.
Enter-PSSession -ContainerId $newcontainer.ContainerId -RunAsAdministrator
```

Finally from the container session notice that the ipconfig.txt file is present.
```
type c:\ipconfig.txt

Ethernet adapter vEthernet (Container-2446380E-6296-4BF2-8146-18DAAFD85FCA-0):

   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::85b:7834:454c:375b%20
   IPv4 Address. . . . . . . . . . . : 192.168.1.55
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . :
```

Finally run the following to stop the remote PowerShell session:

```powershell
exit
```

##Step 4 - Remove Containers and Images

To stop all running containers run the command below. If any containers are in a stopped state when you run this command, you receive a warning, which is ok.
```powershell
Get-Container | Stop-Container
```
To remove all containers:

```powershell
Get-Container | Remove-Container -Force
```
To remove a particular container image, run the following:

```powershell
Get-ContainerImage -Name newimage | Remove-ContainerImage -Force
```

##Host a Web Server in a Container

This next example will walk through a more practical use case for a Windows Server Container. The steps included in this exercise will complete the following:  
- Create a container from the Windows Server Core base image.  
- Deploy web server software into the container.  
- Create a new image from the modified container.  
- Deploy a web server ready container and host a simple website in the container.

##Step 1 – Create Source Container

To create a web server container image, you first need to deploy and start a container from the Windows Server Core base image.
```powershell
$container = New-Container -Name webbase -ContainerImageName WindowsServerCore -SwitchName "Virtual Switch"
 ```

Start the container.
```powershell
Start-Container $container
```

##Step 2 - Install Web Server Software

The next step to prepare the container is to install the web server software. This example will use nginx for Windows. Download and extract the nginx software to 'c:\nginx-1.9.3' on the container. The software can be downloaded from [nginx for Windows](http://nginx.org/en/download.html). Or use the following commands in the container to automatically download and extract the nginx software to 'c:\nginx-1.9.3'.

Establish a remote PowerShell session.
```powershell
Enter-PSSession -ContainerId $container.ContainerId -RunAsAdministrator
```

Download the nginx software.
```
Invoke-WebRequest 'http://nginx.org/download/nginx-1.9.3.zip' -OutFile "c:\nginx-1.9.3.zip"
```

Extract the nginx software.
```
PowerShell.exe Expand-Archive -Path C:\nginx-1.9.3.zip -DestinationPath c:\ -Force
```

Exit the PSSession and stop the container using the following commands.


Exit the remote PowerShell session.
```powershell
exit
```

Finally stop the container using the following command. 
```
Stop-Container $container
```
##Step 3 - Create Web Server Image

With the container updated to include the nginx web server software, you can now create an image from this container. To do so, run the following command:
```powershell
$webserverimage = New-ContainerImage -Container $container -Publisher Demo -Name nginxwindows -Version 1.0
```
When completed, use the `Get-ContainerImage` command to validate that the image has been created.

```powershell
Get-ContainerImage

Name              Publisher    Version      IsOSImage
----              ---------    -------      ---------
nginxwindows      CN=Demo      1.0.0.0      False
WindowsServerCore CN=Microsoft 10.0.10254.0 True
```

##Step 3 - Deploy Web Server Container

To deploy a Windows Server Container based off of the nginxwindows image, use the `New-Container` PowerShell command.

```powershell
$webservercontainer = New-Container -Name webserver1 -ContainerImageName nginxwindows -SwitchName "Virtual Switch"
```
Start the container. and create a remote PowerShell session with the container using `Enter-PSSession`.
```
Start-Container $webservercontainer
```

Create a remote PowerShell session with the new container.
```
Enter-PSSession -ContainerId $webservercontainer.ContainerId -RunAsAdministrator
```

Once working inside the container, the nginx web server can be started and web content staged. To start the nginx web server, change to the nginx installation directory.
```
cd c:\nginx-1.9.3\
```
Start the nginx web server.
```
start nginx
```

When the nginx software is running, get the IP address of the container using `ipconfig`. On a different machine, open up a web browser and browse to `http//container ip`. If everything has been correctly configured, you will see the nginx welcome page.

![](media/nginx.png)

At this point, feel free to update the website. Copy in your own sample website, or run the following command to replace the nginx welcome page with a ‘Hello World’ web page.

```powershell
Invoke-WebRequest 'https://raw.githubusercontent.com/neilpeterson/index/master/index.html' -OutFile "C:\nginx-1.9.3\html\index.html"
```
After the website has been updated, navigate back to `http://container ip`.

![](media/hello.png)

####Resources
[Back to Container Home](../containers_welcome.md)  
[Managing Windows Containers with Docker](./manage_docker.md)