# Automated Minecraft Server Deployment in Linux
## Introduction

Nothing is sweeter than setting up you're very own Minecraft Server for you and your family and friends to play on. But there can be a lot of hassle getting it all setup. Especially in automating the process. In this I will demonstrate all the steps of what to do to set it up. Eventually I will have a script that will do all the install for you, but that is still in the works.

*Note: This is strictly for Minecraft Java Version. Bedrock is coming later. Stay tuned.*

*Note: All of the following steps will be done on a Ubuntu 22.04 LTS, but this can be replicated as far back as Ubuntu 18.04 LTS and Debian 8.*

## Part 1 – Package Installation and Security Updates

To begin, we'll setup a fresh install of Ubuntu/Debian. Like with every new server install, make sure you have the latest package and security updates.

> sudo apt update && sudo apt upgrade -y

Next, we have some packages to install:

> sudo apt install -y openjdk-#-jre screen ufw unattended-upgrades wget

*Note: You'll need to replace <#> with the one appropriate to the version of Minecraft you'll be on.*

| Minecraft Version | Compatible Java Version |
| :------: | :------: |
| <= 1.11 | Java 7 | 
| 1.12 - 1.13 | Java 8 |
| 1.14 - 1.16 | Java 11 | 
| 1.17 - 1.19 | Java 17 |

*Note: You can run Minecraft 1.16 on Java 8, though it is recommended to run the latest verison of Java the Minecraft server supports. But anything past 1.17 you are **required** to run Java 17 or the server will crash.

To explain each of the packages we are installing: 
- Openjdk-#-jre: is the package for the Java software that runs the Minecraft server
- Screen: is a package used by the screen software to run our Minecraft server as a background process.
- UFW: is a package that is used by the UFW (uncomplicated firewall) that we'll use to lock down our Minecraft server ports
- Unattended-upgrades: is the package used for automating security update installation to keep our Minecraft server secure
- Wget: is a package used to download the minecraft server jar files that we'll need

*Note: You don't have to use screen, you can also use tmux or similar. But screen is the one chosen in this tutorial and in the script file that you will download*

## Setup Firewall Rules

In order for your Minecraft server to accessible to greater internet, you'll need to setup Port Forwarding or similar. We will not demonstrate how that is done. But look up articles on Reddit, Facebook, Stackoverflow or you're respective router's company website to find information on how to do so.

Suffice to say, you need to designate a port on your router that is open to the internet that you will use to connect to your Minecraft server. By default, that port is 25565 running on the TCP protocol. Now you can choose to keep this default port, or you can change it. It's up to you. All that it changes is how your Minecraft client connects to the server.

For ease of explaining this process, we will use the default 25565/tcp port for our use case. So we'll need to add that to our firewall rules using UFW.

> sudo ufw allow 25565/tcp

Additionally, you can choose to add an SSH access port if you so desire.

> sudo ufw allow ssh 

or 

> sudo ufw allow 22/tcp

After you have all your firewall rules setup, you'll want to make sure the firewall is enabled and set to automatically run on system start up.

> sudo ufw enable

## Minecraft Server Download

Next, you'll need to head over to Minecraft's official site for the download link for whichever version of Minecraft your going to run. 

https://www.minecraft.net/en-us/download/server

This is the download link for 1.19.2:

https://piston-data.mojang.com/v1/objects/f69c284232d7c7580bd89a5a4931c3581eae1378/server.jar

If you aren't able to find the version you want, they are also archived at this website:

https://mcversions.net/

Now, before we actually download it we need to make a new directory for our Minecraft server files to live in. I like using the /var directory, but it really doesn't matter where you put it as long as you secure it down.

> sudo mkdir /var/minecraft
> <br> cd /var/minecraft

Next, we'll use wget to get our Minecraft server jar file.

> wget <insert_copy_link>

That will pull down the file that should be simliar to "server.jar". 

*Note: The name of the Minecraft server jar file doesn't make a big difference, as long as you change the name of the jar file that the startup file we'll download in a second is looking for. They need to match or the server will fail.*

## Build Server Framework 

Now that we have our server jar file, we need to create a simple bash script that will run that jar and populate the server framework directory including the EULA.txt file that we'll edit.

That script looks like the following (and is included in ![Files](/files/minecraft/minecraft-startup.sh) of this for reference).

> java -Xmx1024M -Xms512M -jar server.jar -nogui

You can create this file using nano, vim, or whatever text editor you prefer. Or you can simply download it from "files". 

Next, we need to run that file to actual populate our framework:

> ./minecraft-startup.sh

After that we need to edit the EULA.txt file to "true". Again, you can use a text editor for this, or you can run the following command:

> echo eula=true > eula.txt

## Create Minecraft System User and Group

For security and ease of automation, we're going to create a user for our Minecraft server to run under. Naturally, we'll call both minecraft, haha.

This can be done in many ways, but I will show you the way I like to use.

> sudo groupadd minecraft
> <br> sudo adduser minecraft --system
> <br> sudo usermod -aG minecraft minecraft

What this is doing is creating a group for minecraft. Creating a system user (can't be logged into), but with the inital group of 'nobody'. We then change it's group to minecraft.





## Set Filesystem Permissions



## Minecraft Security Log4j Warning

If you are running a server between Minecraft 1.7 and 1.18 then it is **absolutely imperative that you follow these directions**. If you are not familiar with Log4j and all of the ramifications from the discovery of that vulnerability, then do yourself a quick favor and do a quick Google search.

This is the link to the official article about it from Minecraft:

https://help.minecraft.net/hc/en-us/articles/4416199399693-Security-Vulnerability-in-Minecraft-Java-Edition

Suffice to say, it is a major problem and could allow **complete take over** of your Minecraft server, and the VM/container that is running it if you do not mitigate this vulnerability.

Lucky for you, Minecraft has a released such mitigation. You can download those files directly from the link above, or I have them included in the files for this tutorial. 

![Log4j v1.12 - v1.26](/files/minecraft/log4j2_112-116.xml)

![Log4j v1.7 - v1.11](/files/minecraft/log4j2_17-111.xml)

I will demonstrate where in the run file for the Minecraft server that you need to include that mitigation, per Minecraft's official instructions. Reference them as you wish.

## Additional Notes

Make note about updating to new versions of Minecraft.

## Resources

- https://www.minecraft.net/en-us/download/server
- https://minecraft.fandom.com/wiki/Tutorials/Setting_up_a_server
- https://mcversions.net/
- https://www.java.com/releases/
- https://help.minecraft.net/hc/en-us/articles/4416199399693-Security-Vulnerability-in-Minecraft-Java-Edition
