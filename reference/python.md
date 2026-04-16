---
title: Python
last_reviewed: 2026-03-07
owner: Rognheim
---

# :fontawesome-brands-python:


## **Python**

---

### **Introduction**

Python is a versatile, high-level programming language known for its simplicity and readability, making it a popular choice for various self-hosting applications. Widely used in web development, automation, and data analysis, Python boasts a rich ecosystem of libraries and frameworks, enabling rapid development and deployment of self-hosted services. With its strong community support, regular updates, and extensive resources, Python continues to be a preferred language for businesses and individuals seeking a flexible and powerful programming tool for their self-hosting needs.

---

### **Installation**

??? Success "On Cloud-Init Debian 12 VM Template"

```bash
sudo apt install python3
```

```bash
sudo apt install python3
```

#

### **Add Python to $PATH**

```bash
which python3
```

```bash
nano ~/.profile
```

```bash
user@dev:~/python$ cat /home/user/.profile 
# ~/.profile: executed by the command interpreter for login shells.
# This file is not read by bash(1), if ~/.bash_profile or ~/.bash_login
# exists.
# see /usr/share/doc/bash/examples/startup-files for examples.
# the files are located in the bash-doc package.

# the default umask is set in /etc/profile; for setting the umask
# for ssh logins, install and configure the libpam-umask package.
#umask 022

# if running bash
if [ -n "$BASH_VERSION" ]; then
    # include .bashrc if it exists
    if [ -f "$HOME/.bashrc" ]; then
	. "$HOME/.bashrc"
    fi
fi

# set PATH so it includes user's private bin if it exists
if [ -d "$HOME/bin" ] ; then
    PATH="$HOME/bin:$PATH"
fi

# set PATH so it includes user's private bin if it exists
if [ -d "$HOME/.local/bin" ] ; then
    PATH="$HOME/.local/bin:$PATH"
fi

# Add python to $PATH
export PATH="/usr/bin/python3:$PATH"
```

```bash
source ~/.profile
```

```bash
echo $PATH
```

#

### **Create an Isolated Environment**

??? Note "It's highly recommended to use a virtual environment to avoid conflicts between dependencies for different projects."

To create and activate a virtual environment:

```bash
python3 -m venv data_entry
```

#

### **Install Pandas**

```bash
pip install pandas
```

You can also list additional dependencies (e.g., NumPy, Matplotlib) in a requirements.txt file and install them using:

```bash
pip install -r requirements.txt
```

#

### **Test**


