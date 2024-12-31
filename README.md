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
12. (Optional) Install `nbgrader` for auto-grading jupyter notebooks:
   	``` bash
    pip install git+https://github.com/samhinshaw/nbgrader.git
   	```
13. (Optional) Install `nbgitpuller` for pulling jupyter notebooks from git repositories:
   	``` bash
    pip install nbgitpuller
    ```
    n/b: `nbgrader` & `nbgitpuller` would be installed in the built docker image for the containers instead.
    
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
17. Modify jupyterhub config file:
	- Set `ltiauthenticator` as `jupyterhub`'s authenticator class in `jupyter_config.py` generated above:
    ``` python

    from ltiauthenticator.lti11.auth import LTI11Authenticator # make sure to match the LTI authenticator version used in canvas/LMS
	c.JupyterHub.authenticator_class = LTI11Authenticator
	c.LTI11Authenticator.allow_all = True # allow authenticated users

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

	- Change `authenticator.allow_existing_users`, `c.LTI11authenticator.allow_all` to True

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
Canvas only adds apps with a configured secured URL so we need to secure your JupyterHub server with a TLS (SSL) certificate and ensure it’s accessible via a domain name using HTTPS.

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
			
			proxy_http_version 1.1;
			proxy_set_header Upgrade $http_upgrade; # allow web socket upgrade
			proxy_set_header Connection "upgrade"; # allow web socket upgrade for jupyter kernel connection
    
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
	- Open a browser and go to `https://<your_domain_name>/hub/lti/launch`.
	You should see a JupyterHub 405: Method not allowed response. This means the https launch URL is working and can now be used to add a jupyterhub app in canvas.
	- 

### Setup Docker & DockerSpawner for Isolated Environments within server (allows multi-users have their isolated environment)
1.	Install Docker:
	- Remove any existing docker versions(if present):
	```bash
	sudo apt-get remove docker docker-engine docker.io containerd runc
	```
    - Install dependencies for apt to use HTTPS:
    ```bash
    sudo apt-get update
	sudo apt-get install ca-certificates curl gnupg
	```
	- Add Docker’s official GPG key:
	```bash
	sudo install -m 0755 -d /etc/apt/keyrings
	curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
	sudo chmod a+r /etc/apt/keyrings/docker.gpg
	```
	- Set up the Docker apt repository:
	```bash
	echo \
	"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
	$(. /etc/os-release && echo "$UBUNTU_CODENAME") stable" | \
	sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
	```
	- Install Docker:
	```bash
	sudo apt-get update
	sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
	```
	- Verify installation:
	```bash
	sudo systemctl status docker
	```
2.	Create a `docker` Group and Add Your User
	- Create a docker group (if for some reason it wasn’t created automatically; check by running ```bash getent group docker```):
	```bash
	sudo groupadd docker
	```
	- Add your user to the docker group:
	```bash
	sudo usermod -aG docker <USERNAME>
	```
	Replace <USERNAME> with the user who will run JupyterHub or needs Docker access
	**Log out and log back in** so that your group membership is re-evaluated.
	
	- Test Docker without sudo:
	```bash
	docker ps
	```
	You should see a list of containers (or an empty list) without any permission errors.

3.	Create Dockerfile and build docker image
	- Create new folder for docker
	```bash
	mkdir dockerdir
	```
	
	- `cd` into the folder and create a docker file
	```bash
	nano Dockerfile
	```
	
	```dockerfile
	# Use a Jupyter Docker Stack as your base.
	# 'jupyter/scipy-notebook' includes conda, numpy, scipy, matplotlib, pandas, etc.
	FROM jupyter/scipy-notebook:latest
	
	# === [1] Switch to root to perform administrative tasks
	USER root
	
	
	# === [2] (Optional): Override the inherited health check from the base image because it looks for a 'server.json' file that we don't have 
	# HEALTHCHECK --interval=30s --timeout=5s --start-period=30s CMD \
	#    /usr/bin/curl --fail http://localhost:8888/api || exit 1
	
	
	# === [2] Upgrade conda, install mamba for faster package solves
	RUN conda update -n base -c defaults conda -y && \
		conda install -n base -c conda-forge mamba -y && \
		conda clean -afy
	
	# === [3] Upgrade pip 
	RUN /opt/conda/bin/pip install  --upgrade pip # if this fails, delete this from dockerfile
	
	# === [4] Install core Python packages using mamba
	# Pin NumPy below 2.0 to avoid compatibility issues
	RUN mamba install --yes \
		"numpy<2.0.0" \
		pandas \
		tensorflow \
		scipy \
		&& mamba clean -afy
	
	# === [5] Install PyTorch and related libraries with CUDA 12.1 support
	RUN mamba install --yes \
		pytorch torchvision torchaudio pytorch-cuda=12.1 -c pytorch -c nvidia \
		&& mamba clean -afy
	
	# === [6] Install additional packages via pip
	# - nbgrader: for assignment creation and grading
	# - nbgitpuller: for distributing notebooks via git
	# - jupyterlab-git: to integrate git into JupyterLab UI
	RUN pip install nbgrader nbgitpuller jupyterlab-git
	
	# update jupyterlab-git to make it compatible
	RUN pip install --upgrade jupyterlab-git nbdime
	 
	# === [7] Enable nbgrader server extensions (still as root to avoid permission issues)
	RUN jupyter server extension enable --sys-prefix --py nbgrader
	
	# === [8](Optional) Create a new user 'newuser' with home directory '/path/to/desired/home/in/container'
	RUN useradd -m -d /path/to/desired/home/in/container -s /bin/bash newuser
	
	# Define environment variables for user 'newuser'
	ENV NB_USER=newuser
	ENV HOME=/path/to/desired/home/in/container
	WORKDIR $HOME
	
	# Change ownership of the home directory to 'newuser'
	RUN chown -R <host_username>:newuser /home/newuser
	
	# === [9] Disable the inherited health check so container won't keep failing
	HEALTHCHECK NONE
	
	#
	
	# === [10] Switch to 'newuser' user
	USER newuser
	
	# (Optional) If you need to install user-level JupyterLab extensions, do so here.
	```
	
	- Build docker Image
	```bash
	docker build --no-cache --progress=plain -t <desired-image-name>:latest .
	```
	
	
4.	Configure DockerSpawner
	- Install DockerSpawner in the virtual environment (created earlier ) sitting on the host machine
	```bash
	pip install dockerspawner
	```
	- Configure Jupyter hub for docker spawner`jupyterhub_config.py`. Docker spawner takes an authenticated user and spawns a notebook server in a Docker container for the user.
	```python
	c.JupyterHub.bind_url = "http://:8000"  # containers interact with jhub via this  
	c.JupyterHub.hub_bind_url = "http://0.0.0.0:8081" # the hub listens to communication via this
	c.JupyterHub.hub_connect_url = "http://172.17.0.1:8081" # single-user containers must connect to the Hub at the Docker host’s IP
	# or older style syntax
	c.JupyterHub.hub_connect_ip = "172.17.0.1"
	# If you are on Docker Desktop for some reason (Mac/Windows), you'd do:
	# c.JupyterHub.hub_connect_url = "http://host.docker.internal:8081"
	
	# DockerSpawner config - set jupyter hub to use docker spawner to allow containers spawn notebook servers 
	c.JupyterHub.spawner_class = "dockerspawner.DockerSpawner"
	
	# The Docker image for single-user containers
	c.DockerSpawner.image = "<image_name>:latest"
	
	c.DockerSpawner.network_name = "bridge" # typically dockers network for the created containers
	c.DockerSpawner.use_internal_ip = True # lets docker assign internal ips to the containers, 172.17.0.X
	
	# If spawns take a while, you can increase timeouts:
	c.Spawner.http_timeout = 120
	c.Spawner.start_timeout = 120
	
	# Define a volume named "jupyterhub-user-{username}" for each user to persist user data after containers are removed/destroyed on user session termination
	c.DockerSpawner.volumes = {
	  'jupyterhub-user-{username}': '/path/to/home'
	}
	
	# Notebook directory inside the container. If your image sets `HOME` to /path/to/home,
	# you might want to keep things consistent.
	c.DockerSpawner.notebook_dir = "/path/to/home"
	
	# Remove containers when they stop (helps keep things clean)
	c.DockerSpawner.remove_containers = True # set false if you need to see container logs which can be used for debugging

	```
	
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
- Edit tool and check box option to allow canvas to open Jupyter hub in a new window
- Test it by launching JupyterHub from within the course module to make sure the LTI integration works.
