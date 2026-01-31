# A checklist for setting up a server

This checklist is used whenever a server gets set up at [Spatie](https://spatie.be).

At the time of writing, we have over 100 active [Digital Ocean](https://www.digitalocean.com) droplets (servers) listed in [Laravel Forge](https://forge.laravel.com). We host a lot of small websites, and host one site per droplet. This ensures that issues on a server for client A won't affect client B's servers. A single, larger, server could be more cost effective, but would add a lot of stress and operational overhead. We made this decision five years ago, and haven't looked back since.

We also host some larger projects (like [Flare](https://flareapp.io)) on AWS provisioned with [Laravel Vapor](https://vapor.laravel.com). Setting up an app on Vapor is outside of the scope of this checklist.

## Support us

[<img src="https://github-ads.s3.eu-central-1.amazonaws.com/checklist-server-setup.jpg?t=1" width="419px" />](https://spatie.be/github-ad-click/checklist-server-setup)

We invest a lot of resources into creating [best in class open source packages](https://spatie.be/open-source). You can support us by [buying one of our paid products](https://spatie.be/open-source/support-us).

We highly appreciate you sending us a postcard from your hometown, mentioning which of our package(s) you are using. You'll find our address on [our contact page](https://spatie.be/about-us). We publish all received postcards on [our virtual postcard wall](https://spatie.be/open-source/postcards).

## 1. Before getting started

Inform a project manager about the server setup. Ensure that it's clear who will bear the costs of the server (Spatie or the client), and what those costs are.

## 2. Provision the server on Laravel Forge

**Log in to [Laravel Forge](https://forge.laravel.com).** The credentials are stored in our 1Password vault.

**Create a new server.** Choose DigitalOcean as the provider. Use the domain name as the server name, replacing dots with dashed. A server for `spatie.be` would be named `spatie-be`.

- Choose the smallest server and the Frankfurt region (most of our clients are based in western Europe)
- Choose the latest PHP version and MySQL 5.7
- Enable DigitalOcean weekly backups

Create the server, and wait a few minutes for it to provision.

**After the new server is provisioned, set up the site.** First, delete the "default" site Forge created.

- Set the site's domain as root domain, for example `spatie.be`
- Set the web directory to `/current/public` when using our zero-downtime deploy script

**Store the server credentials in our 1Password vault.** Forge sends the server's credentials to our technical mailbox. Create a new login in 1Password with the full set of credentials as a note.

## 3. Further provisioning with Ansible

After the basics have been provisioned with Forge we have some additional setup and configuration to do. We've automated these small tasks with Ansible.

Our Ansible playbook will:

- Add all of our active team member's SSH keys to the server's authorized keys
- Add 2GB of swap memory
- Enable passwordless `sudo` commands
- Set the max. upload size in PHP to 30MB
- And more…

You can find the location of our Ansible server in our 1Password vault.

**Add a hosts entry for the new server** in `ansible/hosts` with your preferred editor.

**Install Python on the newly provisioned server.** Python is required to run some provisioning scripts. SSH to the new server from the Ansible droplet and run `sudo apt-get install python`. Then `exit` back to Ansible.

**Provision the new server with Ansible.** Run `ansible-playbook ansible/playbooks/provisionServer.yml --ask-sudo-pass -l spatie-be`. Replace `spatie-be` with the server name you just added to the `hosts` file.

- The provisioning script will ask for the sudo password once. Use the sudo password provided by Forge earlier
- Next, the script will ask for `Name of site directory in /home/forge:`, type the site name you created in the new forge server, for example `spatie.be`

## 4. Deploy the website

After provisioning with Ansible, you should be able to access the server with `ssh forge@123.4.5.6` using the droplet's IP address.

**Create a deploy key on GitHub.** SSH to the server and copy the contents of `~/.ssh/id_rsa.pub`. Use it to create a deploy key on GitHub (Repository > Settings > Deploy keys).

**Create an `.env` file in the site's root directory.** Based on the repository's `.env.example` file, create an `.env` file in the site's root directory, for example `~/spatie.be`.

**Remove the `current` directory.** Envoy needs to create a symlink at this location.

**Double check.** Your site's directory should then look like this:

```
forge@spatie-be:~/spatie.be$ ls
persistent  public  releases .env
```

**Deploy the site.** Assuming the project uses Envoy, run `envoy run deploy` from a local copy of the application to deploy the project for the first time.

**Enable Laravel Horizon.** Log into Forge, and add a daemon to your server. 

- Set the command to `php artisan horizon`
- Set the path to `/home/forge/spatie.be/current`, replacing `spatie.be` with your site's domain

## 5. Set up the demo site

Skip this step if you don't need to set up a demo site on a separate subdomain.

We often set up demo sites on `*.spatie.be`. The demo site may exist until the project goes live, as it runs on the same droplet as the production site. This is not the same as a staging environment!

**Add the demo URL to the NGINX configuration.** In Forge, edit the site's NGINX configuration and add the demo site URL.

```diff
 server {
     listen 80;
     listen [::]:80;
-    server_name spatie.be;
+    server_name spatie-demo.spatie.be spatie.be;
```

**Configure the demo URL's DNS.** It should point to the new server.

```
spatie-demo.spatie.be.  3600 IN A 123.4.5.6
```

## 6. Update the live site's DNS settings

Skip this step for now if you're setting up a server to replace an existing site.

Add an A record with the IP address of the new server, and a CNAME for the `www.`-version.

```
spatie.be.      3600 IN A 123.4.5.6
www.spatie.be.  3600 IN CNAME spatie.be.
```