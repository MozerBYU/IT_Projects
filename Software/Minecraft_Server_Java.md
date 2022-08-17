# Tutorial: Setup Minecraft Java Server in Ubuntu/Debian
## Introduction

![Minecraft Server Image](/Software/assets/images/minecraft/minecraft-server.png)

Nothing is sweeter than setting up your very own Minecraft Server for you, your family and friends to play on. But there can be a lot of hassle getting it all setup. That hassle is exasperated by attempting to automating the process. In this tutorial of sorts I will demonstrate all the steps of what to do to set it up, complete with automating security patches/updates, server startup/shutdown, and server backups. 

Eventually I will have a script that will do all the install for you, but that is still in the works.

*Note: This is strictly for Minecraft Java Version. Bedrock is coming later. Stay tuned.*

*Note: All of the following steps will be done on Ubuntu Server 22.04 LTS , but this can be replicated as far back as Ubuntu 18.04 LTS and Debian 8. It can also be done with the Desktop versions of either Ubuntu or Debian.*

## Part 1 – Package Installation and Security Updates

To begin, we'll setup a fresh install of Ubuntu/Debian. Like with every new server install, make sure you have the latest package and security updates. Ubuntu and Debian use the apt package manager for all of their package repositories. The follow command will update the apt repositories and then search for upgrades for the packages on your system as well as security upgrades/patches for your Operating System.

> `sudo apt update && sudo apt upgrade -y`

Next, we have some additional packages to install that I will explain in a bit:

> `sudo apt install -y openjdk-#-jre screen ufw unattended-upgrades wget`

*Note: The following are optional packages if you are planning to use Github for cloud backups of your server.*

> `sudo apt install -y git`

*Note: You'll need to replace <#> with the one appropriate to the version of Minecraft you'll be on.*

| Minecraft Version | Compatible Java Version |
| :------: | :------: |
| <= 1.11 | Java 7 | 
| 1.12 - 1.13 | Java 8 |
| 1.14 - 1.16 | Java 11 | 
| 1.17 - 1.19 | Java 17 |

*Note: You can run Minecraft 1.16 on Java 8, though it is recommended to run the latest verison of Java the Minecraft server supports. However, anything past 1.11 you are **required** to run Java 8, and anything past 1.16 you are **required** to run Java 17 or the server will crash.*

To explain each of the packages we are installing: 
- Git: is a package for Git that I like for moving files around easily in the commandline
- Openjdk-#-jre: is the package for the Java JDK software that runs the Minecraft server
- Screen: is a package used by the screen software to run our Minecraft server as a background process.
- UFW: is a package that is used by UFW (uncomplicated firewall) that we'll use to lock down our Minecraft server ports except for those we need
- Unattended-upgrades: is the package used for automating security update/patch installation to keep our Minecraft server secure
- Wget: is a package used to download the Minecraft server jar files that we'll need

*Note: You don't have to use screen, you can also use tmux or similar. But screen is the one chosen in this tutorial and in the script file that you will download later.*

## Setup Firewall Rules

In order for your Minecraft server to accessible to greater internet, you'll need to setup Port Forwarding or similar. We will not demonstrate how that is done in this tutorial. If you need help in this area, feel free to look up articles on Reddit, Facebook, StackOverFlow or even your respective router's website to find information on how to do so. If you are still stuck, shoot me a message on Reddit at u/MozerBYU and I'll help you as I have time available.

Suffice to say, you need to designate a port on your router that is open to the internet that you will use to connect to your Minecraft server, and then forward that port to an internal port on your server/computer/VM/container/whatever that is hosting your Minecraft server. This will require you to setup a static IPv4 address on said whatver. By default, that port is 25565 running on the TCP protocol. Now you can choose to keep this default port, or you can change it. It's up to you. All that it changes is how your Minecraft client connects to the server.

For ease of explaining this process, we will use the default 25565/tcp port for our use case. So we'll need to add that to our firewall rules of our server hosting our Minecraft server using UFW. Like is said in the name, UFW is designed to uncomplicate the firewalling process as much as possible. If you want to use IPtables or similar, go for it. However, for simplicity, we're going to stick with UFW in this tutorial. The follow command will tell UFW to allow our Minecraft server to be open to inbound connections.

> `sudo ufw allow 25565/tcp`

Additionally, you can choose to add an SSH access port if you so desire.

> `sudo ufw allow ssh`

or 

> `sudo ufw allow 22/tcp`

After you have all your firewall rules setup, you'll want to make sure the firewall is enabled and set to automatically run on system start up.

> `sudo ufw enable`

## Minecraft Server Download

Next, you'll need to head over to Minecraft's official site for the download link for whichever version of Minecraft your going to run. 

https://www.minecraft.net/en-us/download/server

This is the download link for 1.19.2 (latest as of 8/15/22):

https://piston-data.mojang.com/v1/objects/f69c284232d7c7580bd89a5a4931c3581eae1378/server.jar

If you aren't able to find the version you want, they are also archived at this website:

https://mcversions.net/

Now, before we actually download the jar file, we need to make a new directory for our Minecraft server files to live in. I like using the /var directory, but it really doesn't matter where you put it as long as you secure it down.

> `sudo mkdir /var/minecraft`
> <br> `cd /var/minecraft`

Next, we'll use wget to get our Minecraft server jar file.

> `wget <insert_copy_link>`

That will pull down the file that should be simliar to "server.jar". 

*Note: The Minecraft server jar filename doesn't make a big difference, as long as you change the name of the jar file that the startup file we'll download in a second is looking for. They need to match or the server will fail.*

## Create Initial Server Framework 

Now that we have our server jar file, we need to create a simple bash script that will run that jar and populate the initial server framework directory including the EULA.txt file that we'll edit.

That script looks like the following below:

> `java -Xmx1024M -Xms512M -jar server.jar -nogui`

*Note: Don't worry about tweaking these values, as this is just to setup the framework of our Minecraft server. We'll tweak the actual memory values later for the server.* 

You can create this file using nano, vim, or whatever text editor you prefer. Or you can simply download it here:

![Minecraft Startup Script](/Software/assets/files/minecraft/java/minecraft-startup.sh)

Next, we need to run that file to actual populate our framework:

> `./minecraft-startup.sh`

After that we need to edit the EULA.txt file to "true". Again, you can use a text editor for this, or you can run the following command:

> `echo eula=true > eula.txt`

## Create Minecraft System User and Group

For security and ease of automation, we're going to create a user for our Minecraft server to run under. Naturally, we'll call both minecraft, haha.

This can be done in many ways, but I will show you the way I like to use.

> `sudo groupadd minecraft`
> <br> `sudo adduser minecraft --system`
> <br> `sudo usermod -aG minecraft minecraft`

What this is doing is creating a group for minecraft. Creating a system user (that can't be logged into), but with the inital group of 'nobody'. We then change it's group to minecraft. That way if something goes awry with your Minecraft server, it is more difficult to affect the rest of the system.

## Import World if needed

The cool thing about creating servers, is you can create a new fresh install (and even set the seed). Or you can use existing "world" files. 

If you do choose a new world, you set the world seed in the "server.properties" file.

As far transferring existing world files, your method will depend on what version of Ubuntu/Debian you're running (whether or not it has a GUI or not). If you do have a GUI, then you can simply use Google Drive, Dropbox or the equivalent. If you're like me and are using the server edition, the easiest method I have found for moving the file around in the commandline is by using Git. You can use a Git server of your choice, even your own if you have one. In my case, I choose to use Git LFS provided by Github, as it is specifically designed for large files and some of my Minecraft world files are quite large.

**If you opt to not use git, you can ignore this whole section.**

*Note: I won't be covering very much about git in this tutorial. Just the basics of creating a repo, adding the world files, pushing it up to the cloud. And then pulling back down on the desired machine.*

You may be wondering why I've chosen to use Git, and not something else. I like Git because it does versioning. Which is how I a lot of my cloud backups for my Minecraft servers. Granted, I have several backups running. These are *some* of my off-shore backups that I have in place. 

First on your local computer, install Git. Next, you'll want to create a directory for your Git repository to live in (which will put that "world" folder in later). To do this you'll run the following commands:

> `git init -b main`
> <br> `git add .`
> <br> `git -m "Initial commit"` *(Don't ask me why, but it has to be 'Initial commit' or **it will break**)*

Now, you need to go to GitHub and create a repository for your Minecraft server to live in. Once that is complete we can then tell git to push our server files up to the cloud.

> `git remote origin add https://<git server repository>`
> <br> `git remote -v`
> <br> `git push origin main`

Once you have your git repository created and the world file added to that repository, we're ready to clone it down. Which can be done by using git clone.

> `sudo git clone https://<git_server>/<user>/<repository_for_minecraft>`
  
From there, just make sure that your world folder is in the right location in /var/minecraft.

## Set Filesystem Permissions

Now a very important note. Most of the work we have been doing has either been as root or a sudo user. But our Minecraft server will be running as the minecraft user. So we need to change some permissions on /var/minecraft or it will break when you try to run it. Using the following command will allow us to setup the correct permissions on the /var/minecraft folder and everything in it.
  
> `sudo chown -R minecraft:minecraft /var/minecraft`

## Minecraft Security Log4j Warning

If you are running a server between Minecraft 1.7 and 1.18 then it is **absolutely imperative that you follow these directions**. If you are not familiar with Log4j and all of the ramifications from the discovery of that vulnerability, then do yourself a quick favor and do a quick Google search.

This is the link to the official article about it from Minecraft:

https://help.minecraft.net/hc/en-us/articles/4416199399693-Security-Vulnerability-in-Minecraft-Java-Edition

Suffice to say, it is a major problem and could allow **complete take over** of your Minecraft server, and the VM/container/system that it is running on if you do not mitigate this vulnerability.

Lucky for you, Minecraft has released a mitigation for it. You can download those files directly from the link above, or I have them included in the files for this tutorial. 

![Log4j v1.12 - v1.16 XML File](/Software/assets/files/minecraft/java/log4j2_112-116.xml)

![Log4j v1.7 - v1.11 XML File](/Software/assets/files/minecraft/java/log4j2_17-111.xml)

I will demonstrate where in the run file for the Minecraft server that you need to include that mitigation, per Minecraft's official instructions. Reference them as you wish.
  
Below is our original run script for the Minecraft Server:
  
> `java -Xmx1024M -Xms512M -jar server.jar -nogui`
  
Here is the edited version with the Log4j mitigation in place:

> `java -Xmx1024M -Xms512M -Dlo4j.configurationFile=log4j2_<version>.xml -jar server.jar -nogui`

*Note: for the next section I have a file that has the mitigation included.*

## Server Automation using Systemd

Now, we have everything in place. However, there is a bit of problem. Everytime your server/VM/container kicks off and has to restart, you would originally need to re-run that minecraft-startup.sh file manually. But that can get hectic if you don't have a VPN or other remote access option available. This my friend is where the beauty of service files comes in.

Service files allow you to designate a certain process or service to automatically run on startup of the system, as well as other features like auto-restarting and shutdown.

The script I provide is a modified and simplified verison of the original script found here:

https://minecraft.fandom.com/wiki/Tutorials/Server_startup_script

I edited it for simplicities sake for single Minecraft server instance installs. I will include the original script down below if you wish to use the other, with some of my modifications necessary in order to make it actually run. Yes, you're welcome world. It took me a while to figure that out.

![Simplified Minecraft Service File](/Software/assets/files/minecraft/java/minecraft.service)

![Original-Edited Minecraft Service File](/Software/assets/files/minecraft/java/minecraft-original.service)

Now if you need the script with Log4j mitigation in place, that can be found here (adjust as needed for your version). Don't forget to also download the appropriate log4j migitigation configuration xml file.

![Log4j Minecraft Service File](/Software/assets/files/minecraft/java/minecraft-log4j.service)

Don't forget to edit the -Xmx and -Xms values to the desired levels for your server. I will say, 512M is the minimum amount of memory you should allocate for -Xms. Anything lower, and Java will complain a shiz ton to you about not having enough memory. In my personal experience, Minecraft server versions >= 1.18 need a minimum of 2.5GB or 2560M for -Xmx in order to run correctly.

*Note: Yes, these scripts work with Bukkit and Paper. I am running 5 Paper Minecraft servers right now using this exact script. I just went and edited the section that has the -jar to be appropriate to name of the jar file for your Paper server.*

Once you have the script downloaded, make sure to title it minecraft.service and put it in the /etc/systemd/system directory so that it can be found by systemd. You can do that by using the following command:

> `sudo mv minecraft-<whatever>.service /etc/systemd/system/minecraft.service`

Now to test and make sure everything is working to this point:

> `sudo systemctl start minecraft`

Followed by:

> `sudo systemctl status minecraft`

or

> `sudo service minecraft start`

Followed by:

> `sudo service minecraft status`

If you see that the world has finished creating/processed and there are no errors then we are good to enable our Minecraft Service file using either of the following commands:

> `sudo systemctl enable minecraft`

or 

> `sudo service minecraft enable`

## Miscellanous History

**Skip if you don't care about the history of why I edited the public script.**

History time, so back about 2 years ago (2021), I was in a spot where I was using tmux and manually going in everytime to start my Minecraft servers using `bash minecraft-startup.sh`. Note, I was running 5 Minecraft servers at that time, each with their own VM/container. So I had to repeat this process 5 times, everytime I had a server go down or had to run maintenance or cloud backups.

As you can imagine this painstaking process was rather frustrating. So I did a good bit of research on StackOverFlow, Minecraft Fandom and r/Minecraft and r/FeedtheBeast (aka ModdedMinecraft). Which eventually led me to learn about Linux Service files as a means of automating running a service, or process, on system startup. 

After some more digging I came across that article I referenced above. I was so excited to copy it and get to work. However, soon thereafter I ran into some problems. This original script is for deploying several Minecraft servers to a given machine via minecraft-x@service. Which in my case didn't really apply, as I have each Minecraft server on a seperate VM/container. Additionally, there were some issues with how the script was setup where the server would die in a matter of seconds, but then endless attempt to restart itself to then die again. On top of that, the error/debug messages were so ambiguous (don't ask, it's been over a year and a half and I don't remember, nor care now). 

Suffice to say, one error was related to the instruction "Type=forking". Which after a lot of research, I modified to be "Type=simple". As I noted in my modification, the following StackOverFlow discussion has a fairly good explanation of the issue and what let me to change the original instruction: 

https://askubuntu.com/questions/953920/systemctl-service-timed-out-during-start

That modification solved the issue of the server dying at instant speed and endless trying to restart. But then I had another issue encountered in the "ExecStart" section of this beast:

![Original ExecStart Script](/Software/assets/images/minecraft/original-execstart-script.PNG)

This had to be modified as well, by myself, so that screen was invoked by the process prior to invoking Java. Otherwise, screen was called after Java and it would crash the heck out of the server. Also, as I will get to later, none of the backups worked properly.

![Modified ExecStart Script](/Software/assets/images/minecraft/modified-execstart-script.PNG)

However, like I mentioned, I didn't need the functionality of create multiple Minecraft server instances on a given VM/container. So I stripped away that functionality for ease of reading and ease of use with the script for my own purposes. If you're setup is simliar to mine then you'll want the simplified script. If not, then feel free to use the modified version of the original script.

![Simplified ExecStart Script](/Software/assets/images/minecraft/simplified-execstart-script.PNG)
          
## Server Backups  
You can rely on automated backups, which I'll demonstrate just below thanks to the work of others I'll credit here in a bit. Or if you feel the painful option you can always do a manual one by doing the following:
  
> `cp /var/minecraft/world /var/minecraft/world.backup`

There are many ways to do backups of your Minecraft server. Various plugins exist just for the purpose. And many scripts have been created for such a purpose. The issue that arises with doing **live** backups of your Minecraft server, is active writes to the server. Say you have 5 players that are on all building something while the backup takes place, the backup could be inconsistent, it could endup being corrupted, or it could turn out fine. It's hard to tell what will be the result. 

But it is better to be safe than sorry. The script provided by Nicholas Chan on GitHub (https://github.com/nicolaschan/minecraft-backup) has proven to be very helpful in properly backing up Minecraft servers. The way it works it is connects to the actual console of the Minecraft server (via screen, tmux or similar) and then issues a `/save-off` command to tell the server to stop any writes. After the backup has completed it will issue a `/save-on` command to the server to resume any writes, and save any pending writes that occurred during the backup.

Feel free to use whatever backup method suits you best. I will demonstrate how to use the script referenced above.

To begin head over and download the script to whichever directory suits you best (i.e. /var/scripts/). Next, you'll want to ensure that the script is executable by running the following command:

> `sudo chmod u+x /var/script/backup.sh`

Once you ahve the script download and the permissions set open a program called 'crontab' by running the following:

> `crontab -e`

From here you can edit the file with whatever text editor you prefer. Basically at the end of the file you need to add the schedule of when you want cron to run a job to execute our backup script for the Minecraft server. 

The format for cron is the following: 

| Minute | Hour | Day (month) | Month | Day (week) |
| :------: | :------: | :------: | :------: | :-------: |

If you need help figuring out to set it up, we found this helpful tool online to assist you:

https://crontab.guru/

For my setup, I have crontab run the backups every day at 1:00am. So mine looks like the following:

> 0 1 * * * /var/scripts/backup.sh -c -i /var/minecraft/world -o /var/minecraft-backups -s MC

To explain what is happening above. We tell cron when to run the script we are calling, backup.sh in this case. We give it an input directory (the world directory for our Minecraft server) and and output directory (for the backups) and then tell it to use screen using -s and what 'screen' to look for. In the Minecraft Service file, we designate that screen as MC.

*Note: If you change that screen name from MC to anything else in that service file you will need to change it here as well, or it won't work.*

One last thing I will mention about backups. Make sure you save them somewhere (cloud, external hdd, flashdrive, whatever) other than you system. Yes, still have backups on the system itself. But in case that system completely dies, you'll want offshore backups. Trust me. It has saved a server world of mine a time or two. Like I said earlier, GitHub has been super helpful in that area, as I use it for monthly backups and other major backups as necessary (like after a large creative world build).

Good luck and let me know if you have any questions or comments! I'll update this again when I get the automated deployment script created.

## Additional Notes

### Server Updating
  
A word of caution. Before you do **any** upgrade from one major version to another, make sure to do your research. With minor verisons, like 1.91.1 to 1.19.2 there usually isn't an issue. But regardless, always, *always* make a backup of your world before doing an upgrade. Granted, backups of your Minecraft server should be part of your routine maintenance, which were shown previously.
 
## Troubleshooting
  
Errors with starting the Minecraft server can be attributed to a simple subset of a few things:
  
- 1) You have the Minecraft service file looking for the wrong directory
- 2) You have the wrong permissions set on /var/minecraft
- 3) You forgot to create the user and group for minecraft
- 4) You have the wrong version of Java appropriate to your verison of Minecraft

Checking these should allievate most errors you encounter in new Minecraft server setups. Feel free to ping me at r/MozerBYU if you have unique issues or you get stuck banging your head against a wall after several hours of troubleshooting and researching.
  
## Resources

- https://www.minecraft.net/en-us/download/server
- https://minecraft.fandom.com/wiki/Tutorials/Setting_up_a_server
- https://mcversions.net/
- https://www.java.com/releases/
- https://help.minecraft.net/hc/en-us/articles/4416199399693-Security-Vulnerability-in-Minecraft-Java-Edition
- https://docs.github.com/en/get-started/importing-your-projects-to-github/importing-source-code-to-github/adding-locally-hosted-code-to-github
- https://minecraft.fandom.com/wiki/Tutorials/Server_startup_script
