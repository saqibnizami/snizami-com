---
date: 2019-06-26
title: Configure Jupyter Notebook for Windows Subsystem for Linux 2
---
# Using Jupyter Notebook with WSL2

Due to the archetectural changes in Windows Subsystem for Linux 2, the linux filesystem is hosted on a virtual drive. This VHD is accessed through an IP assigned to WSL2. To access the a Jupyter Notebook server running in WSL2, you must set the Jupyter Notebook config file to allow connections from locations other than localhost. 

## Generate Jupyter Notebook Config File

If you do not have a `jupyter_notebook_config.py` file in your `~/.jupyter` directory, run `jupyter notebook --generate-config` in your Terminal. 

## Edit the Config File

Connections from outside `localhost` must be allowed by changing two values in the config file.

Change `c.Notebook.App.allow_origin = "*"` to allow all origins
Change `c.Notebook.App.ip = '0.0.0.0'` to allow all IPs

## Unblock port

As a precaution, unblock the default jupyter port: `sudo ufw allow 8888`

## Set a password for Jupyter Notebook

Run `jupyter notebook password` and enter a password when prompted

## Find the WSL2 IP address
Open the WSL2 Terminal (bash/ubuntu/windows terminal) and run `ip addr | grep eth0 | grep inet`
Use this IP to access jupyter notebook, i.e. `https://<IP>:8888`

## Start Jupyter Notebook

`jupyter notebook` and connect using the wsl ip
