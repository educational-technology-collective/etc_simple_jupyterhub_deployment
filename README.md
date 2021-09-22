# Simple JupyterHub Deployment

This document gives instructions for how to deploy JupyterHub and JupyterLab in a "bare metal" or VPS multiuser server environment.  These step by step instuctions are intended to help the administrator understand how the deployment works and how the components work together.  

These instructions are adapted from [Install JupyterHub and JupyterLab from the ground up](https://github.com/jupyterhub/jupyterhub-the-hard-way/blob/abe84bfb4418c00e08effd1486e2b666fb803ac8/docs/installation-guide-hard.md).

## Requirements

- Basic understanding of networking, the command line interface, working with configuration files, and `systemd` services.
- `sudo` access to a server or VPS.
- Ubuntu 18.04.1 LTS.
    - The instructions assume you are working from an Ubuntu system; however, you may be able to adapt the instructions to other Linux distributions given a sufficient understanding of package management.
- SSL certificate.
    - The instructions provide guidance on configuring the JupyterHub endpoint to use SSL.

## Assumptions

The instructions in this document assume a common and reasonable server configuration.

## Goals

You will deploy a JupyterHub instance that is managed by `systemd` and contained in a `conda` managed environment.  This deployment will use the default PAM Authenticator for authentication.  

Servers commonly contain multiple Python installations. Each Python environment may serve a particular purpose.  Hence, one of the goals of this deployment is for JupyterHub to be installed in a `conda` environment separate from other environments on the system.  This approach will help ensure the stability of the deployment.  Likewise, using `conda`, or another environment manger, each user will be able to configure their own environment to meet their specific needs.

### The steps of this deployment are:

- [Install the Configurable HTTP Proxy](#install-the-configurable-http-proxy)
    - You will install the Configurable HTTP Proxy, which will serve as an endpoint for the JupyterHub installation.
- [Install the `conda` Package Manager](#install-the-conda-package-manager)
    - You will install `conda` into `/opt/conda` in order to manage our JupyterHub environment and default user environment. 
- [Create a `conda` Environment for JupyterHub and Install JupyterHub](#create-a-conda-environment-for-jupyterhub-and-install-jupyterhub)
    - You will create a `conda` environment specifically for JupyterHub and install JupyterHub into it.
- [Create a Default `conda` Environment for JupyterLab Users](#create-a-default-conda-environment-for-jupyterlab-users)
    - You will create a default environment that JupyterLab users can use in order to run Notebooks.  You will make it available to JupyterLab.
- [Configure JupyterHub](#configure-jupyterhub)
    - You will configure JupyterHub.
- [Configure a `systemd` Service for JupyterHub](#configure-a-systemd-service-for-jupyterhub)
    - You will configure `systemd` to manage the JupyterHub deployment and start JupyterHub.
- [Make User Installed Python Environments Available to JupyterLab](#make-user-installed-python-environments-available-to-jupyterhub)
    - You will learn how to install and configure kernel specs in order to make environments installed by users visible to JupyterLab.

## Install the Configurable HTTP Proxy

JupyterHub is composed of 3 major subsystems: [the Hub, the Proxy, and Single-User Notebook Servers](https://jupyterhub.readthedocs.io/en/stable/reference/technical-overview.html#the-subsystems-hub-proxy-single-user-notebook-server).  JupyterHub starts a Configurable HTTP Proxy when it starts.  It serves two primary functions: 

- It forwards requests to JupyterHub e.g., it acts as an endpoint.
- When JupyterHub spawns a new Single User Notebook Server, it permits JupyterHub to configure it in order to forward url prefixes to the newly spawned server.

Furthermore, having the proxy act as an endpoint, JupyterHub can be restarted without interrupting connections between users and their already spawned Notebook servers.

### Install `nodejs` and `npm`.

```bash
sudo apt install nodejs npm
```

### Install `configurable-http-proxy`.

```bash
sudo npm install -g configurable-http-proxy
```

JupyterHub will start and configure the proxy when it starts.

##  Install the `conda` Package Manager

You will use `conda` in order to create and manage the JupyterHub deployment and create additional environments for use in JupyterLab. `conda` will be used in order to create and manage a default environment that will be made available to users of JupyterLab.  Likewise, users *may* use `conda` in order to manage their custom environments.

Instructions for installing conda using a RPM or Debian package manager are described in [RPM and Debian Repositories for Miniconda](https://docs.conda.io/projects/conda/en/latest/user-guide/install/rpm-debian.html).

These instructions will install `conda` into the `/opt` directory, which is designated for "[Add-on application software packages](https://www.pathname.com/fhs/pub/fhs-2.3.html#OPTADDONAPPLICATIONSOFTWAREPACKAGES)".  Because `conda` is an independently published package and it will not be used by the system, the `/opt` directory is a suitable directory for its installation.

The following instuctions are adpated from [RPM and Debian Repositories for Miniconda](https://docs.conda.io/projects/conda/en/latest/user-guide/install/rpm-debian.html).  These actions will install `conda` into `/opt/conda` on an Ubuntu server.  Instructions for other Linux distributions are also available.

### Install `conda`.

```bash
# Install our public GPG key to trusted store.
curl https://repo.anaconda.com/pkgs/misc/gpgkeys/anaconda.asc | gpg --dearmor > conda.gpg
install -o root -g root -m 644 conda.gpg /usr/share/keyrings/conda-archive-keyring.gpg

# Check whether fingerprint is correct (will output an error message otherwise).
gpg --keyring /usr/share/keyrings/conda-archive-keyring.gpg --no-default-keyring --fingerprint 34161F5BF5EB1D4BFBBB8F0A8AEB4F8B29D82806

# Add our Debian repo.
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/conda-archive-keyring.gpg] https://repo.anaconda.com/pkgs/misc/debrepo/conda stable main" | sudo tee -a /etc/apt/sources.list.d/conda.list

# Make the newly added repository available to `apt`.
sudo apt update

# Install `conda`.
sudo apt install conda
```

## Create a `conda` Environment for JupyterHub and Install JupyterHub

You will use `conda` in order to create a new environment for JupyterHub in `/opt/jupyterhub` and install JupyterHub and JupyterLab into the environment.  This environment will be created and managed by the root account.  Because `/opt/jupyterhub` is not a default path that `conda` searches for environments, this environment is hidden from users i.e., it won't show up in `conda env list`.

### Create the environment and install JupyerHub and required components.

```bash
sudo /opt/conda/condabin/conda create --prefix /opt/jupyterhub python=3.8 jupyterhub jupyterlab ipykernel
```

The installation of `ipykernel` will result in the creation of a [Kernel spec](https://jupyter-client.readthedocs.io/en/latest/kernels.html#kernelspecs), which we will discuss below.

This environment can be managed by using the `sudo /opt/conda/condabin/conda` command with `--prefix /opt/jupyterhub`.

Examples

#### List the packages installed in this environment.

```bash
sudo /opt/conda/condabin/conda list --prefix /opt/jupyterhub
```

#### Install a package into the environment.

```bash
sudo /opt/conda/condabin/conda install --prefix /opt/jupyterhub the-name-of-the-package
```

## Create a Default `conda` Environment for JupyterLab Users

You will use `conda` in order to create a default environment named `jupyterhub_default` that will be available to users of JupyterLab.  Users that haven't installed custom environments will use this environment in order to run their Notebooks.  This environment will be created and managed by the root account.  However, it will be installed into the `/opt/conda/envs` directory, which `conda` will search by default when searching for environments; hence, this environment will show up in `conda env list` and users can activate it; however, only root can modify it.

This environment will serve as a default environment available to all users of JupyterLab.  Instructions for creating user specified environments will be provided in a following section.

### Create the environment and install Python.

```bash
sudo /opt/conda/condabin/conda create --name jupyterhub_default python=3.8 
```

### Manage the environment.

This environment can be managed by using the `sudo /opt/conda/condabin/conda install -n jupyterhub_default` command.  For example, additional packages can be installed into this environment using:

```bash
sudo /opt/conda/condabin/conda install -n jupyterhub_default the-name-of-the-package
```

### Make the jupyterhub_default environment and its kernel available to JupyterLab.

The kernels that are provided on the JupyterLab Launcher Panel and the kernels that can be selected in the JupyterLab kernel selector dialog are defined in [Kernel spec *directories*](https://jupyter-client.readthedocs.io/en/latest/kernels.html#kernelspecs).  For this deployment the following kernel spec directories will be searched when JupyterLab starts:

|Priority|Type|Locations|
|---|---|---|
|1|System|/usr/share/jupyter/kernels</br>/usr/local/share/jupyter/kernels|
|2|JupyterHub Environment|/opt/jupyterhub/share/jupyter/kernels|
|3|User Defined|~/.local/share/jupyter/kernels|

A kernel spec is named according to the name of its *directory*, which may reside in one or more of the directories in the table above.  You may have multiple kern specs installed on your system.  A kernel spec is defined in a JSON file, named kernel.json, that may look something like this:

```json
{
 "argv": [
  "/opt/conda/envs/jupyterhub_default/bin/python",
  "-m",
  "ipykernel_launcher",
  "-f",
  "{connection_file}"
 ],
 "display_name": "Python 3 Default (ipykernel)",
 "language": "python",
 "metadata": {
  "debugger": true
 }
}
```

Kernel specs that have the same directory name and that have a higher priority (see the Priority number in the above table) take precedence over lower priority kernel specs of the same name.  Hence, if two kernel specs are provided in `/usr/share/jupyter/kernels/python3` and `/opt/jupyterhub/share/jupyter/kernels/python3` then the latter will override the former.

When you installed JupyterHub and ipykernel, a kernel spec should have been installed in `/opt/jupyterhub/share/jupyter/kernels/python3`.  By default this kernel spec points to the kernel installed in its environment.  However, you want for the `jupyterhub_default` kernel to be offered to users instead of the `jupyterhub` kernel.  Hence, change the first element of the `argv` setting to point to the `jupyterhub_default` kernel by changing the value to `/opt/conda/envs/jupyterhub_default/bin/python`.

#### Open the kernel spec for the `jupyterhub` environment using a text editor and modify it to reference the kernel in the `jupyterhub_default` environment.

```bash
nano /opt/jupyterhub/share/jupyter/kernels/python3/kernel.json
```

The modified kernel.json should look like this.
```json
{
 "argv": [
  "/opt/conda/envs/jupyterhub_default/bin/python",
  "-m",
  "ipykernel_launcher",
  "-f",
  "{connection_file}"
 ],
 "display_name": "Python 3 Default",
 "language": "python",
 "metadata": {
  "debugger": true
 }
}

```

Depending on your environment you may need to remove or override kernel specs contained in one or more of the other kernel spec locations in order to prevent JupyterLab from offering them to your users.  If you are seeing kernel specs in the Launcher Panel that shouldn't be there, examine each of the kernel spec locations in order to address the issue in a way that is appropriate to your specific circumstances.

## Configure JupyterHub

In order to configure the JupyterHub deployment you will create a configuration file.  This section will point out common configuration options.  You may need to make further configuration settings depending on the specific requirements of your system environment.

### Create directories for configuration file and service data files.

```bash
sudo mkdir -p /opt/jupyterhub/etc/jupyterhub/
sudo mkdir -p /opt/jupyterhub/srv/jupyterhub/
```

### Generate the default configuration file.

```bash
cd /opt/jupyterhub/etc/jupyterhub/
sudo /opt/jupyterhub/bin/jupyterhub --generate-config
```

### Open the configuration file using a text editor in order to make changes to the default configuration.

```bash
sudo nano jupyterhub_config.py
```

### Set the location of service data files.

```bash
c.JupyterHub.cookie_secret_file = '/opt/jupyterhub/srv/jupyterhub/jupyterhub_cookie_secret'
c.JupyterHub.db_url = '/opt/jupyterhub/srv/jupyterhub/jupyterhub.sqlite'
c.JupyterHub.pid_file = '/opt/jupyterhub/srv/jupyterhub/jupyterhub-proxy.pid'
```

### Configure JuptyerHub to direct browsers to JupyterLab.

```bash
c.Spawner.default_url = '/lab'
```

### Configure JupyterHub to configure the proxy to use port 8443 and the `/jupyter` prefix as the endpoint.

This setting specifies the address and port that the Configurable HTTP Proxy will bind to.  Modify the URL IP address according to your needs.

```bash
c.JupyterHub.bind_url = 'https://127.0.0.1:8443/jupyter'
```

### Configure the URLs that JupyterHub and the Configurable HTTP Proxy will use to communicate on.

```bash
c.JupyterHub.hub_bind_url = 'http://127.0.0.1:8081'
c.ConfigurableHTTPProxy.api_url = 'http://127.0.0.1:8001'
```

### Provide JupyterHub with the location of your certificate and key file. 

```bash
c.JupyterHub.ssl_cert = '/etc/ssl/private/the-name-of-the-certificate.cert'
c.JupyterHub.ssl_key = '/etc/ssl/private/the-name-of-the-key.key'
```

## Configure a `systemd` Service for JupyterHub

You will configure a `systemd` that can be used in order to manage JupyterHub.

### Create a directory that will hold the `systemd` unit file. 

```bash
sudo mkdir -p /opt/jupyterhub/etc/systemd
```

### Create a `systemd` unit definition file for JupyterHub.
Here we will use the nano text editor in order to create and open the file.
```bash
sudo nano /opt/jupyterhub/etc/systemd/jupyterhub.service
```

### Paste the unit definition into the file and save the file.

```bash
[Unit]
Description=JupyterHub
After=syslog.target network.target

[Service]
User=root
Environment="PATH=/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/opt/jupyterhub/bin"
ExecStart=/opt/jupyterhub/bin/jupyterhub -f /opt/jupyterhub/etc/jupyterhub/jupyterhub_config.py

[Install]
WantedBy=multi-user.target
```

### Create a link to the unit definition file in order for `systemd` to be able to find it.

```bash
sudo ln -s /opt/jupyterhub/etc/systemd/jupyterhub.service /etc/systemd/system/jupyterhub.service
```

### Reload the `systemd` configuration files, enable the service, start the service, and check its status.

```bash
sudo systemctl daemon-reload
sudo systemctl enable jupyterhub.service
sudo systemctl start jupyterhub.service
sudo systemctl status jupyterhub.service
```

JupyterHub should be accessible at the endpoint specified by the `c.JupyterHub.bind_url` setting in the configuration file.

## Make User Installed Python Environments Available to JupyterHub

The easiest way for an user to make an environment visible to JupyterLab is to install `ipykernel` into the environment and then use `ipykernel` in order to create the kernel spec.  The kernel spec will be created in `~/.local/share/jupyter/kernels`; this is one of the paths that JupyterLab searches for kernel specs.

### Make a kernel visible to JupyterLab.

```bash
# Activate the environment.
conda activate the-name-of-the-environment
# Create the kernel spec.
python -m ipykernel install --user --name 'the-name-of-the-kernel-spec' --display-name "The Display Name"
```

- The `--user` argument specifies that the kernel spec should be installed for the current user.  
- The `--name` argument specifies the name of the kernel spec that will be saved to `~/.local/share/jupyter/kernels`.  
- The `--display-name` argument specifies the name that is displayed in JupyterLab.

