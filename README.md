# JupyterHub Setup Guide

This guide provides step-by-step instructions for setting up a server environment for JupyterHub (tested on a scalable DigitalOcean Droplet).


## Setup: JupyterHub (Tested in DigitalOcean Droplet - 4GB 2VCPUs)
### Steps

1. Create a Scalable Server**
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
    virtualenv -ppython3 env
    ```
9.  Enter `env` environment:
    ```bash
    source env/bin/activate
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
13. Install `nbgrader` for pulling jupyter notebooks from git repositories:
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
    ```
- Add the following lines :
	```python
	import os
	from dotenv import load_dotenv # use dotenv to load keys and secret
	load_dotenv()
	```
- Uncomment the allowed_users and add a list of users on the server to allow to access jupyterhub 

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
    sudo su root
    sudo echo 127.0.0.1 $(hostname) >> /etc/hosts
    ```
21. Switch back to new user and start `jupyterhub` with specified `jupyterhub_config.py`:
    ``` bash
    sudo su newuser
    jupyterhub -f /path/to/jupyterhub_config.py
    ```
    
## Configure Canvas for JupyterHub Integration

1. Go to canvas

2. **Log in to Canvas and Go to Admin Settings**
 - Navigate to **Admin** > **Settings** > **Apps** > **View App Configurations**.

3. Add JupyterHub as an External Tool:

- Click **+ App** to add a new tool.
- Set the configuration as follows:
	- **Configuration Type**: Paste XML.
	- **Name**: JupyterHub (or a name that makes sense to your students).
	- **Consumer Key** and **Shared Secret**: Use the ones you configured in jupyterhub_config.py.
	- **XML Configuration**: Use the following XML template and replace YOUR_JUPYTERHUB_URL with your actual JupyterHub URL:
	```xml
	<cartridge_basiclti_link xmlns="http://www.imsglobal.org/xsd/imslticc_v1p0">
  		<blti:title>JupyterHub</blti:title>
  		<blti:description>Access JupyterHub from Canvas.</blti:description>
  		<blti:launch_url>https://YOUR_JUPYTERHUB_URL/hub/lti/launch</blti:launch_url>
  		<blti:extensions platform="canvas.instructure.com">
    		<lticm:property name="privacy_level">public</lticm:property>
  		</blti:extensions>
	</cartridge_basiclti_link>
	```
4. **Add JupyterHub to a Course Module**:
- In Canvas, go to your course and add JupyterHub as an **External Tool** in a module.
- Test it by launching JupyterHub from within the course module to make sure the LTI integration works.
