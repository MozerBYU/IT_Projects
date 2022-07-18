# Cloudflare Argo Tunnel
## Introduction

Cloudflare Argo Tunnel is a very unique software that allows you to setup a secure tunnel between your server and Cloudflare that you can send internet traffic down without exposing your local IP address or any ports. It is very robust with lots of features to suit many needs and configurations.

The following documentation will detail how to setup the Argo Tunnel and the cloudflared daemon which will run the Argo Tunnel autonomously, using the systemd daemon.

## Part 1 – Download Cloudflared

To download the latest cloudflared daemon, run the following command:

> `wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb`

And then install by running the following:

> `sudo dpkg -i cloudflared-linux-amd64.deb`

## Part 2 – Authenticate with Cloudflare

In order to setup the tunnel properly, you’ll need to create an account with Cloudflare, and setup a domain with them (this can be done by purchasing a new domain, or by transferring an existing domain).
After cloudflared has been installed, we need to authenticate with our Cloudflare account so that we can get our cert.pem credentials file so that the tunnel will use to verify that it is our tunnel and not someone elses.

This is done by running the following command:

> `cloudflared tunnel login`

This will then present you with a url to open in your browser to authenticate. After that is complete, you will get a cert.pem file that you will use in all of your config.yml files for your tunnels.

## Part 3 – Create an Argo Tunnel

In this step, we will create a basic tunnel named ‘basic’. You can create additional tunnels if you want later on, but we'll just make one to demonstrate how it is done.

*Note: You can name the tunnel’s whatever you’d like, just make sure to update that info across your configuration files and service files that we’ll show here in a second.*

We create any tunnel by running the following command:

> `cloudflared tunnel create <INSERT_NAME>`

This will give a UUID for your tunnel that you will use when you create associated DNS records for your tunnel, which I will show in the next section.

*Note: You'll want to copy that somewhere for ease of reference later on.*

## Part 4 – Create the associated DNS Records

On the https://dash.cloudflared.com dashboard for your given site, if you go to the DNS section, you can create the necessary DNS records for your domain. For your newly created Argo Tunnel that record is designated as follows:

| Entry | Options |
| :------: | :------: |
| DNS Record Type	| AAAA or CNAME |
| Name	| subdomain or @ |
| Target |	UUID.cfargotunnel.com |
| Proxy Status | Proxied |
| TTL	| Auto or whatever you want (i.e. 5m) |

Now that we have cloudflared installed, the tunnel authenticated and the necessary DNS records created, we have one more step before we are ready to setup cloudflared as a service. And that is creating our first configuration file.

## Part 5 – Install Cloudflared to run as a Service
  
Run the following command in order to have the cloudflared package install all the necessary configuration files:

> `sudo cloudflared –config </path/to/config_file.yml> service install`
  
If all was done correctly, you should be able to go in your web browser and input that subdomain you created and have it proxy your web traffic to what you set in your configuration file.

To verify it is all running as expected, you can go to your log files for your respective webserver and check the access file for the respective domain/subdomain to see that the traffic is coming through correctly. If it is you will see the host IP address as 127.0.0.1, which means that it is coming from the cloudflared daemon through the argo tunnel we just created.

Additionally, to see a better picture of what is happening, you can also run the following:
  
> `sudo lsof -i -P -n`
  
In here you can see all the web connections, what host it is originating from and what port they are using. You should see a connection from your cloudflared daemon to the Cloudflare Edge, as well as a connection from the cloudflared daemon to your webserver, in this case Nginx.

Or if you prefer a prettier browser-based experience you can go to:
  
https://dash.teams.cloudlfared.com
  
From there you’ll go to ‘Access’ and then to ‘Tunnels’. That will allow you to see the tunnel and to see if it is either active or inactive, and what DNS routes are using it.

## Resources

Cloudflare Dashboard
- https://dash.cloudflare.com

Tunnel Creation and Setup
-	https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup

Cloudflare Ingress Rules
-	https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/configuration/configuration-file/ingress
