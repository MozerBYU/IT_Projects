# NGINX WAF Installation Tutorial
## Introduction
The following documentation will detail the various steps to setup a Nginx Web Server, complete with a Web Application Firewall (WAF) using ModSecurity and the OWASP CRS Rule Sets. All of which is compiled documentation from nginx.org official documentation in an easier more concise format. Links to the official documentation can be found in "Resources" at the bottom.

*Note: All parts of this documentation will be done using Ubuntu 20.04. Adjust as needed according to needs and system preferences.*
 
## Part 1 – Setup Nginx with ModSecurity WAF
### Part 1a – Download Nginx and Necessary Packages

First and foremost, don’t use the default apt repository provided by Ubuntu for Nginx, it is several versions behind the mainline release line. Instead you can either a) download the source code directly from Nginx’s official website, or b) you can update the repository for the Nginx package. For sake of this documentation we will be using Nginx version 1.20.2

*Note: you need at least Nginx 1.11.5 in order to use ModSecurity.*

For option a) run the following:

> `wget http://nginx.org/download/nginx-1.20.2.tar.gz`
> <br> `tar zxvf nginx-1.20.2.tar.gz`

For option b) run the following to install the prerequisites and download the nginx signing key:  

> `sudo apt install curl gnupg2 ca-certificates lsb-release ubuntu-keyring`
> <br> `curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor \`
> <br>    `| sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null`

Next verify the file has the right key by running the following:

> `gpg --dry-run --quiet --import --import-options import-show /usr/share/keyrings/nginx-archive-keyring.gpg`

Output should look similar to what is below (taken from Nginx’s official website):

> `pub rsa2048 2011-08-19 [SC] [expires: 2024-06-14]`
> <br> &nbsp; `573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62`
> <br> `uid                   nginx signing key signing-key@nginx.com`

Run the following to setup apt with the latest stable Nginx repo:

> `echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \`
> <br> ``http://nginx.org/packages/ubuntu `lsb_release -cs` nginx" \``
> <br> `| sudo tee /etc/apt/sources.list.d/nginx.list`

And then run the following to pin the repo:

> `echo -e "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" \`
> <br> `| sudo tee /etc/apt/preferences.d/99nginx`

Once that is all complete, we can finally proceed with installing Nginx using apt.

> `sudo apt update`
> <br> `sudo apt install nginx-core`

### Part 1b – Download ModSecurity 3.0 and Compile It

For this next section we’ll need to install some packages that are required for building and compiling ModSecurity.

> `sudo apt install -y apt-utils autoconf automake build-essential git libcurl4-openssl-dev libgeoip-dev liblmdb-dev libpcre++-dev libtool libxml2-dev libyajl-dev pkgconf wget zlib1g-dev`

Next, we’ll download ModSecurity 3.0 from its official repository on GitHub.

> `git clone --depth 1 -b v3/master --single-branch https://github.com/SpiderLabs/ModSecurity`

After that we’ll change into our ModSecurity directory and compile the source code.

> `cd ModSecurity`
> <br> `git submodule init`
> <br> `git submodule update`
> <br> `./build.sh`
> <br> `./configure`
> <br> `make`
> <br> `make install`
> <br> `cd ..`

You may see a bunch of errors messages, don’t worry about it as long as the overall compilation succeeds. 

### Part 1c – Download and Configure the Nginx Connector for ModSecurity

For the next part we are going to download the connector of sorts that will be used by Nginx to connect the ModSecurity Module to Nginx.

> `git clone --depth 1 https://github.com/SpiderLabs/ModSecurity-nginx.git`

Now we will be using that Nginx package we downloaded earlier. This next batch of commands will compile the dynamic module into Nginx.

> `cd nginx-1.20.2`
> <br> `./configure --with-compat --add-dynamic-module=../ModSecurity-nginx`
> <br> `make modules`
> <br> `cp objs/ngx_http_modsecurity_module.so /etc/nginx/modules`
> <br> `cd ..`

### Part 1d – Load the Nginx ModSecurity Connector Module

Open the Nginx configuration file located in /etc/nginx/nginx.conf using your preferred text editor and add the following line, so that Nginx will properly load the ModSecurity module:

> `load_module modules/ngx_http_modsecurity_module.so;`

### Part 1e – Configure and Enable ModSecurity WAF

In this part we are going to create a new directory, modsec, within our default Nginx configuration directory for our ModSecurity configuration files. We will be using the recommended configuration directly for the creators of ModSecurity (SpiderLabs).

> `mkdir /etc/nginx/modsec`
> <br> `wget -P /etc/nginx/modsec/ https://raw.githubusercontent.com/SpiderLabs/ModSecurity/v3/master/modsecurity.conf-recommended`
> <br> `mv /etc/nginx/modsec/modsecurity.conf-recommended /etc/nginx/modsec/modsecurity.conf`

Occasionally there is an error where the unicode.mapping file isn’t copied correctly, if this happens go to the GitHub repo and download it (should be in the master branch). And then simply run the following command to put it in the correct place:

> `cp ModSecurity/unicode.mapping /etc/nginx/modsec`

While we are still in the configuring stage, we need to ensure that ModSecurity is in ‘detection-mode’ only, otherwise it will drop traffic (which can pose problems during the configuration stage). **MAKE SURE TO CHANGE THIS LATER.**

> `sed -i 's/SecRuleEngine DetectionOnly/SecRuleEngine On/' /etc/nginx/modsec/modsecurity.conf`

Now we need to add our rules to our main config file for ModSecurity in /etc/nginx/modsec/main.conf. Ensure that you have the following within that file:

> `# From https://github.com/SpiderLabs/ModSecurity/blob/master/`
> <br> `# modsecurity.conf-recommended`
> <br> `# Edit to set SecRuleEngine On`
> <br> `Include "/etc/nginx/modsec/modsecurity.conf"`

### Part 1f – Add the Free OWASP CRS Rule Set to your ModSecurity Rules

For this part we’re going to download the latest rule set from OWASP’s Github repo.

> `wget https://github.com/SpiderLabs/owasp-modsecurity-crs/archive/v3.0.2.tar.gz`
> <br> `tar -xzvf v3.0.2.tar.gz`
> <br> `sudo mv owasp-modsecurity-crs-3.0.2 /usr/local`

Create the CRS main config file (you can just copy and rename the example file provided).

> `cd /usr/local/owasp-modsecurity-crs-3.0.2`
> <br> `sudo cp crs-setup.conf.example crs-setup.conf`

Next, we’re going to edit our config file for ModSecurity in /etc/nginx/modsec/main.conf to include our new OWASP CRS rule set.

> `# Include the recommended configuration`
> <br> `Include /etc/nginx/modsec/modsecurity.conf`
> <br> 
> <br> `# OWASP CRS v3 rules`
> <br> `Include /usr/local/owasp-modsecurity-crs-3.0.2/crs-setup.conf`
> <br> `Include /usr/local/owasp-modsecurity-crs-3.0.2/rules/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf`
> <br> `Include /usr/local/owasp-modsecurity-crs-3.0.2/rules/REQUEST-901-INITIALIZATION.conf`
> <br> `Include /usr/local/owasp-modsecurity-crs-3.0.2/rules/REQUEST-905-COMMON-EXCEPTIONS.conf`
> <br> `Include /usr/local/owasp-modsecurity-crs-3.0.2/rules/REQUEST-910-IP-REPUTATION.conf`
> <br> `Include /usr/local/owasp-modsecurity-crs-3.0.2/rules/REQUEST-911-METHOD-ENFORCEMENT.conf`
> <br> `Include /usr/local/owasp-modsecurity-crs-3.0.2/rules/REQUEST-912-DOS-PROTECTION.conf`
> <br> `Include /usr/local/owasp-modsecurity-crs-3.0.2/rules/REQUEST-913-SCANNER-DETECTION.conf`
> <br> `Include /usr/local/owasp-modsecurity-crs-3.0.2/rules/REQUEST-920-PROTOCOL-ENFORCEMENT.conf`
> <br> `Include /usr/local/owasp-modsecurity-crs-3.0.2/rules/REQUEST-921-PROTOCOL-ATTACK.conf`
> <br> `Include /usr/local/owasp-modsecurity-crs-3.0.2/rules/REQUEST-930-APPLICATION-ATTACK-LFI.conf`
> <br> `Include /usr/local/owasp-modsecurity-crs-3.0.2/rules/REQUEST-931-APPLICATION-ATTACK-RFI.conf`
> <br> `Include /usr/local/owasp-modsecurity-crs-3.0.2/rules/REQUEST-932-APPLICATION-ATTACK-RCE.conf`
> <br> `Include /usr/local/owasp-modsecurity-crs-3.0.2/rules/REQUEST-933-APPLICATION-ATTACK-PHP.conf`
> <br> `Include /usr/local/owasp-modsecurity-crs-3.0.2/rules/REQUEST-941-APPLICATION-ATTACK-XSS.conf`
> <br> `Include /usr/local/owasp-modsecurity-crs-3.0.2/rules/REQUEST-942-APPLICATION-ATTACK-SQLI.conf`
> <br> `Include /usr/local/owasp-modsecurity-crs-3.0.2/rules/REQUEST-943-APPLICATION-ATTACK-SESSION-FIXATION.conf`
> <br> `Include /usr/local/owasp-modsecurity-crs-3.0.2/rules/REQUEST-949-BLOCKING-EVALUATION.conf`
> <br> `Include /usr/local/owasp-modsecurity-crs-3.0.2/rules/RESPONSE-950-DATA-LEAKAGES.conf`
> <br> `Include /usr/local/owasp-modsecurity-crs-3.0.2/rules/RESPONSE-951-DATA-LEAKAGES-SQL.conf`
> <br> `Include /usr/local/owasp-modsecurity-crs-3.0.2/rules/RESPONSE-952-DATA-LEAKAGES-JAVA.conf`
> <br> `Include /usr/local/owasp-modsecurity-crs-3.0.2/rules/RESPONSE-953-DATA-LEAKAGES-PHP.conf`
> <br> `Include /usr/local/owasp-modsecurity-crs-3.0.2/rules/RESPONSE-954-DATA-LEAKAGES-IIS.conf`
> <br> `Include /usr/local/owasp-modsecurity-crs-3.0.2/rules/RESPONSE-959-BLOCKING-EVALUATION.conf`
> <br> `Include /usr/local/owasp-modsecurity-crs-3.0.2/rules/RESPONSE-980-CORRELATION.conf`
> <br> `Include /usr/local/owasp-modsecurity-crs-3.0.2/rules/RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf`


### Part 1g – Reload/Restart Nginx and Test our Configuration

*Note: As a side note for the future, to do a quick test of Nginx configuration files to ensure they are all working properly with the correct syntax we can run a simple command to check that:*

> `nginx -t`

If all looks good, we are finally ready to reload/restart Nginx and test to make sure everything was setup properly. An easy way to do this can be done with the Nikto Scanning Tool. Running the following commands with download the tool and run a scan on localhost.

> `git clone https://github.com/sullo/nikto`
> <br> `cd nikto`
> <br> `perl program/nikto.pl -h localhost`

*Note: It will take some work to configure the rules to your specific needs and to fine tune them in general to remove false positives. The following article has excellent advice on how to do this:*

https://docs.nginx.com/nginx-waf/admin-guide/nginx-plus-modsecurity-waf-owasp-crs/

The following note applies if you chose option b in the beginning to update the apt repository for Nginx.

*Note: As with most production environments, use caution when updating your server and it's packages as this may update Nginx to a new version before it has been re-compile with ModSecurity for that version of Nginx. Which will break your installation of Nginx.*
 
## Resources

Update Apt for Official Nginx Repo
-	http://nginx.org/en/linux_packages.html#Ubuntu
ModSecurity WAF Setup 
-	https://www.nginx.com/blog/compiling-and-installing-modsecurity-for-open-source-nginx
OWASP CRS Rules
-	https://docs.nginx.com/nginx-waf/admin-guide/nginx-plus-modsecurity-waf-owasp-crs/
Rebuilding/Recompiling Nginx
-	https://www.nginx.com/blog/failed-nginx-plus-upgrade-module-version-x-instead-of-y
