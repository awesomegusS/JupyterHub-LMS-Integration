# JupyterHub Setup Guide

This guide provides step-by-step instructions for setting up a server environment for JupyterHub (tested on a scalable DigitalOcean Droplet).


## Setup: JupyterHub (Tested in DigitalOcean Droplet - 4GB 2VCPUs)
### Setup Server Environment with JupyterHub

1. Provision a Server/VPS machine**
- Set up a scalable server for the installation.

2. Spin Up the Server and Connect to it
- Launch the server and ensure it is accessible.

3. Install Updates
Update the installed packages and dependencies:
	```bash
	sudo apt update && sudo apt upgrade
	```

4. Install `python`, `virtual env`:
	```bash
	sudo apt install python3 python3-venv
	```
5. Add a new system user:
	```bash
	sudo adduser newuser
	```
6. Edit new user `sudoers` file:
- Edit the `sudoers` file to allow specific privileges:
	```bash
	sudo visudo
	```
	- Add the following line at the end of the file:
	```sql
	newuser ALL=(ALL:ALL) ALL
	```
	replace `newuser` with name of `newuser` created
7. Switch to new user:
	```bash
	sudo su newuser
	```
8. Create virtualenv `env` for running sandboxed `jupyterhub`:
   	```bash
    python3 -m venv jupyterhub-env
    ```
9.  Enter `env` environment:
    ```bash
    jupyterhub-env/bin/activate
   	```
10. Install `jupyterhub` for multiuser-session jupyter notebooks:
	``` bash
    pip install jupyterhub
    ```
11. Install `jupyterhub-ltiauthenticator` for multiuser-session jupyter notebooks:
   	``` bash
    pip install jupyterhub-ltiauthenticator
    ```
12. Install `nbgrader` for auto-grading jupyter notebooks:
   	``` bash
    pip install git+https://github.com/samhinshaw/nbgrader.git
   	```
13. Install `nbgitpuller` for pulling jupyter notebooks from git repositories:
   	``` bash
    pip install nbgitpuller
    ```
14. Generate `LTI_CLIENT_KEY` and `LTI_CLIENT_SECRET` then save to `.env`:
   	``` bash
    LTI_CLIENT_KEY=(`openssl rand -hex 32`)
    LTI_CLIENT_SECRET=(`openssl rand -hex 32`)
    echo LTI_CLIENT_KEY=$LTI_CLIENT_KEY > .env
    echo LTI_CLIENT_SECRET=$LTI_CLIENT_SECRET >> .env
    source .env
   	```
   
15. Install `python-dotenv` for environment variable usage in python script:
   	``` bash
    pip install python-dotenv
    ```
16. Generate config file for `jupyterhub`:
   	``` bash
    jupyterhub --generate-config
   	``` 
17. Modify config file:
- Set `ltiauthenticator` as `jupyterhub`'s authenticator class in `jupyter_config.py` generated above:
    ``` python
    #c.JupyterHub.authenticator_class = 'jupyterhub.auth.PAMAuthenticator'
    import os
    c.JupyterHub.authenticator_class = 'ltiauthenticator.LTIAuthenticator'
    c.LTIAuthenticator.consumers = {
        os.environ['LTI_CLIENT_KEY']: os.environ['LTI_CLIENT_SECRET']
    }
    c.LTIAuthenticator.username_key = 'lis_person_sourcedid'
    ```
- Add the following lines :
	```python
	import os
	from dotenv import load_dotenv # use dotenv to load keys and secret
	load_dotenv()
	```

- Change `authenticator.allow_existing_users` to True

18. Install `npm` and `nodejs`:
	```bash
	sudo apt install -y nodejs npm
	```
	
19. Install `configurable-http-proxy` as it is needed by `jupyterhub`:
    ```bash
    sudo npm install -g configurable-http-proxy
    ```
    
20. Switch back to root and add machine's hostname to `/etc/hosts`:
    ``` bash
    su - root
    sudo echo 127.0.0.1 $(hostname) >> /etc/hosts
    ```
21.	Switch back to new user and start `jupyterhub` with specified `jupyterhub_config.py`:
    ``` bash
    su - newuser
    jupyterhub -f /path/to/jupyterhub_config.py
    ```

### Setup HTTPS for app(jupyterhub) launch URL/domain name
Canvas only adds apps with a configured with a secured URL so we need to secure your JupyterHub server with a TLS (SSL) certificate and ensure it’s accessible via a domain name using HTTPS.

1.	Obtain a Domain Name
	If you don’t already have one, purchase a domain from a domain registrar and configure its DNS to point to your server’s IP address. For instance, set an A record for jupyter.example.com to the server’s IP.
	You can use a subdomain services such as duckdns.org, netlify.app, vercel.app, github.io, etc.

2.	Install and start nginx (for reverse proxying)
	It’s considered best practice to run JupyterHub behind a reverse proxy (e.g., Nginx or Traefik) that handles HTTPS termination. This way, JupyterHub can continue running on HTTP internally (localhost:8000), while the proxy provides a secure HTTPS interface to the outside world.
	- Install Nginx on your server (for reverse proxying)
	```bash
	sudo apt update && sudo apt install nginx
	```
	- Ensure Nginx is running:
	```bash
	sudo systemctl start nginx
	```
3.	Obtain a TLS Certificate (Let’s Encrypt)
	Use Certbot to automatically obtain and renew free certificates from Let’s Encrypt:
	- Install certbot and obtain TLS certs for your domain name:
	```bash
	sudo apt install certbot python3-certbot-nginx
	sudo certbot --nginx -d jupyter.example.com
	```
	Certbot will prompt you to select options and will automatically configure Nginx for HTTPS. After successful completion, you’ll have a valid certificate for `https://jupyter.example.com`
	
4.	Configure Nginx as a Reverse Proxy for JupyterHub
	If Certbot didn’t automatically create a proxy configuration for JupyterHub, you can manually set it up. Create or edit an Nginx server block (e.g., `/etc/nginx/sites-available/jupyter`):
	- Create the config file
	```bash
	sudo nano /etc/nginx/sites-available/jupyter
	```
	```nginx
	server {
    	listen 80;
    	server_name jupyter.example.com;
    	return 301 https://$host$request_uri;
	}

	server {
    	listen 443 ssl;
    	 http2 on; # Enable HTTP/2
    	server_name jupyter.example.com;

		ssl_certificate /etc/letsencrypt/live/jupyter.example.com/fullchain.pem;
		ssl_certificate_key /etc/letsencrypt/live/jupyter.example.com/privkey.pem;
		include /etc/letsencrypt/options-ssl-nginx.conf;
		ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
	
		location / {
			proxy_pass http://127.0.0.1:8000;
			proxy_set_header Host $host;
			proxy_set_header X-Real-IP $remote_addr;
			proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
			proxy_set_header X-Forwarded-Proto $scheme;
		}
	}

	```
	Replace `jupyter.example.com` with your domain name
	- Disable the default `nginx` config by removing the symbolic link in `/etc/nginx/sites-enabled`:
	```bash
	sudo rm /etc/nginx/sites-enabled/default
	```
	
	- Enable your new Jupyter config by creating a link to in `/etc/nginx/sites-enabled` to point to existing jupyter config file in `/etc/nginx/sites-available`:
	```bash
	sudo ln -s /etc/nginx/sites-available/jupyter /etc/nginx/sites-enabled/jupyter
	```
	
	- Test and Reload `nginx`:
	```bash
	sudo nginx -t
	sudo systemctl reload nginx
	```
5.	Run JupyterHub 
	```bash
    su - newuser
    jupyterhub -f /path/to/jupyterhub_config.py
	```

6.	Test the HTTPS Launch URL for JupyterHub
	- Open a browser and go to `https://jupyter.example.com/hub/lti/launch`.
	You should see a JupyterHub 405: Method not allowed response. This means the https launch URL is working and can now be used to add a jupyterhub app in canvas.
	- 
	
    
## Configure Canvas for JupyterHub Integration

1. Go to canvas

2. **Log in to Canvas and Go to Admin Settings**
 - Navigate to **Admin** > **Settings** > **Apps** > **View App Configurations**.

3. Add JupyterHub as an External Tool:

- Click **+ App** to add a new tool.
- Set the configuration as follows:
	- **Configuration Type**: Manual Entry.
	- **Name**: JupyterHub (or a name that makes sense to your students).
	- **Consumer Key** and **Shared Secret**: Use the ones you configured in jupyterhub_config.py.
	- **Launch URL**: `https://jupyter.example.com/hub/lti/launch` use your registered domain name URL
	- **Domain**: use your registered domain `jupyter.example.com`
	
4. **Add JupyterHub to a Course Module**:
- In Canvas, go to your course and add JupyterHub as an **External Tool** in a module.
- Test it by launching JupyterHub from within the course module to make sure the LTI integration works.
