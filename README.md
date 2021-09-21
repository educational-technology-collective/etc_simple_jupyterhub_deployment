# Simple JupyterHub Deployment

This document gives instructions for how to deploy JupyterHub and JupyterLab in a "bare metal" or VPS multiuser server environment.  These step by step instuctions are intended to help the administrator understand how the deployment works and how the components work together.  

These instructions are adapted from [Install JupyterHub and JupyterLab from the ground up](https://github.com/jupyterhub/jupyterhub-the-hard-way/blob/abe84bfb4418c00e08effd1486e2b666fb803ac8/docs/installation-guide-hard.md).

## Requirements

- Basic understanding of networking, the command line interface, working with configuration files, and `systemd` services.
- `sudo` access to a server or VPS.
- Ubuntu 18.04.1 LTS.  The instructions assume you are working from an Ubuntu system; however, you may be able to adapt the instructions to other Linux distributions given a sufficient understanding of package management.

## Assumptions

The instructions in this document assume common and reasonable server environments.

## Goals

These instructions will describe how to deploy a JupyterHub instance that is managed by `systemd` and contained in a `conda` environment.  

Servers commonly contain multiple Python installations. Each Python environment may serve a particular purpose.  Hence, one of the primary goals of this deployment is for JupyterHub to be installed in a `conda` environment separate from other environments on the system.  This approach will help ensure the stability of the deployment.  Likewise, using `conda`, or another environment manger, each user will be able to configure their own environment to meet their specific needs.

### The steps of this deployment are:

- [Install the Configurable HTTP Proxy](#install-the-configurable-http-proxy)
    - You will install the Configurable HTTP Proxy, which will serve as an endpoint for the JupyterHub installation.
- [Install the `conda` Package Manager](#install-the-conda-package-manager)
    - You will install `conda` into `/opt/conda` in order to manage our JupyterHub environment and default user environment. 
- [Create a `conda` Environment for JupyterHub and Install JupyterHub](#create-a-conda-environment-for-jupyterhub-and-install-jupyterhub)
    - You will create a `conda` environment specifically for JupyterHub and install JupyterHub into it.
- [Create a Default `conda` Environment for JupyterLab Users](#create-a-default-conda-environment-for-jupyerlab-users)
    - You will create a default environment that JupyterLab users can use in order to run Notebooks.  You will make it available to JupyterLab.
- [Make the jupyterhub_default Environment and its Kernel Available to JupyterLab](#make-the-jupyterhub_default-environment-and-its-kernel-available-to-jupyterlab)
    - You will make the `jupyterhub_default` environment available to JupyterLab.
- [Configure JupyterHub](#configure-jupyterhub)
    - You will configure JupyterHub.
- Configure `systemd` to manage the JupyterHub deployment and Start JupyterHub.
- Create any number of user environments and make them available to the JupyterHub instance.

## Install the Configurable HTTP Proxy

JupyterHub is composed of 3 major subsystems: [the Hub, the Proxy, and Single-User Notebook Servers](https://jupyterhub.readthedocs.io/en/stable/reference/technical-overview.html#the-subsystems-hub-proxy-single-user-notebook-server).  JupyterHub spawns a Configurable HTTP Proxy when it starts.  It serves two primary functions: 

1. It forwards requests to JupyterHub e.g., it acts as an endpoint.
2. When JupyterHub spawns a new Single User Notebook Server, it permits JupyterHub to configure it in order to forward url prefixes to the newly spawned server.

Furthermore, having the proxy act as an endpoint, JupyterHub can be restarted without interrupting connections between users and their already spawned Notebook servers.

Install `nodejs` and `npm`:

```bash
sudo apt install nodejs npm
```

Install `configurable-http-proxy`:

```bash
sudo npm install -g configurable-http-proxy
```

##  Install the `conda` Package Manager

You will use `conda` in order to create and manage our JupyterHub deployment and create additional environments for use in JupyterLab. `conda` will be used in order to create and manage a default environment that will be made available to users of JupyterLab.  Likewise, users *may* use `conda` in order to manage their custom environments.

Instructions for installing conda using a RPM or Debian package manager are described in [RPM and Debian Repositories for Miniconda](https://docs.conda.io/projects/conda/en/latest/user-guide/install/rpm-debian.html).

These instructions will install `conda` into the `/opt` directory, which is designated for "[Add-on application software packages](https://www.pathname.com/fhs/pub/fhs-2.3.html#OPTADDONAPPLICATIONSOFTWAREPACKAGES)".  Because `conda` is an independently published package and it will not be used by the system, the `/opt` directory is a suitable directory for its installation.

The following instuctions are adpated from [RPM and Debian Repositories for Miniconda](https://docs.conda.io/projects/conda/en/latest/user-guide/install/rpm-debian.html).  These actions will install `conda` into `/opt/conda` on an Ubuntu server.  Instructions for other Linux distributions are also available. 

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

```bash
sudo /opt/conda/condabin/conda create --prefix /opt/jupyterhub python=3.8 jupyterhub jupyterlab ipykernel
```

The installation of `ipykernel` will result in the creation of a [Kernel spec](https://jupyter-client.readthedocs.io/en/latest/kernels.html#kernelspecs), which we will discuss below.

This environment can be managed by using the `sudo /opt/conda/condabin/conda` command with `--prefix /opt/jupyterhub`.  For example we can list the packages installed into this environment with:

`sudo /opt/conda/condabin/conda list --prefix /opt/jupyterhub`

## Create a Default `conda` Environment for JupyterLab Users

We will use `conda` in order to create a default environment named `jupyterhub_default` that will be available to users of JupyterLab.  A user can use this environment in order to run their Notebooks.  This environment will be created and managed by the root account.  However, it will be installed into the `/opt/conda/envs` directory, which `conda` will search by default when searching for environments; hence, this environment will show up in `conda env list` and users can activate it; however, only root can modify it.

```bash
sudo /opt/conda/condabin/conda create --name jupyterhub_default python=3.8 
```

This environment can be managed by using the `sudo /opt/conda/condabin/conda install -n jupyterhub_default` command.  For example, additional packages can be installed into this environment using:

```bash
sudo /opt/conda/condabin/conda install -n jupyterhub_default the-name-of-the-package
```

This environment will serve as a default environment available to all users of JupyterLab.  Instructions for creating user specified environments will be provided in a following section.

## Make the jupyterhub_default Environment and its Kernel Available to JupyterLab

The kernels that are provided on the Launcher Panel and that can be selected in JupyterLab are defined in [Kernel spec *directories*](https://jupyter-client.readthedocs.io/en/latest/kernels.html#kernelspecs).  For this configuration the following kernel spec directories will be searched when JupyterLab starts:

|Priority|Type|Locations|
|---|---|---|
|1|System|/usr/share/jupyter/kernels</br>/usr/local/share/jupyter/kernels|
|2|JupyterHub Environment|/opt/jupyterhub/share/jupyter/kernels|
|3|User Defined|~/.local/share/jupyter/kernels|

A kernel spec is named according to the name of its *directory*, which may reside in one of more of the directories in the table above.  A kernel spec is defined in a JSON file, named kernel.json, that may look something like this:

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
Kernel specs that have the same directory name and that have a higher priority take precedence over lower priority kernel specs of the same name.  Hence, if two kernel specs are given in `/usr/share/jupyter/kernels/python3` and `/opt/jupyterhub/share/jupyter/kernels/python3` then the latter will override the former.

When you installed JupyterHub and ipykernel, a kernel spec should have been installed in `/opt/jupyterhub/share/jupyter/kernels/python3`.  By default this kernel spec points to the kernel installed in its environment.  However, you want for the `jupyterhub_default` kernel to be offered to users instead of the `jupyterhub` kernel.  Hence, change the first element of the `argv` setting to point to the `jupyterhub_default` kernel by changing the value to `/opt/conda/envs/jupyterhub_default/bin/python`.

Depending on your environment you may need to remove or override kernel specs contained in one or more of the other kernel spec locations in order to prevent JupyterLab from offering them to your users.  If you are seeing kernel specs in the Launcher Panel that shouldn't be there, examine each of the kernel spec locations in order to address the issue appropriate to your specific circumstances.


