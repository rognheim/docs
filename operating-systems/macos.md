---
title: macOS
last_reviewed: 2026-03-07
owner: Rognheim
---

# :simple-apple:


## **macOS**

---

### **Introduction**

macOS is a proprietary Unix operating system, derived from OPENSTEP for Mach and FreeBSD, which has been marketed and developed by Apple since 2001. It is the current operating system for Apple's Mac computers. Within the market of desktop and laptop computers, it is currently the second most widely used desktop OS, after Microsoft Windows and ahead of all Linux distributions

---

### **Preventing .DS_Store files on SMB shares**

A .DS_Store file is a hidden, proprietary file created by Apple's macOS to store custom attributes of a folder, such as icon positions, view options, and other metadata. These files are not human-readable and are automatically re-created by the Finder unless you prevent it. They can be safely deleted for local folders, but their presence on shared or version-controlled folders can be a nuisance, as they are not intended for cross-platform use

To prevent macOS from creating these files on network shares and external USB devices run the following commands:

```bash
defaults write com.apple.desktopservices DSDontWriteNetworkStores -bool TRUE
```

```bash
defaults write com.apple.desktopservices DSDontWriteUSBStores -bool TRUE
```

### **Removing any .DS_Store files from an already mounted SMB share**

**Safely finding and (optionally) deleting `.DS_Store` files**

You’ll want to:

- Run the search **first** and confirm what will be deleted.
- Then run a controlled delete.
- Ideally, target only the mounted SMB share, not your entire system.

Assume your SMB share is mounted at something like `/Volumes/myshare`. Adjust the path as needed.

#### Step 1: List `.DS_Store` files only (no delete)

```bash
find /Volumes/myshare -name .DS_Store -print
```

Review the output. Confirm:
- The path is correct (only your SMB share).
- The files listed are what you expect.

If your path includes spaces, quote it:

```bash
find "/Volumes/My Share Name" -name .DS_Store -print
```

#### Step 2 (optional but safer): Confirm with a dry-run delete command

Use `-delete` in a *dry run* with `echo` to be sure:

```bash
find /Volumes/myshare -name .DS_Store -print0 | xargs -0 echo rm
```

This will just *print* the `rm` commands that would be run, without executing them.

Check that:
- No unexpected paths show up.
- Only `.DS_Store` files on that share are listed.

#### Step 3: Delete `.DS_Store` files

Once you’re confident:

**Method A: Using `-delete` (simple, recommended)**

```bash
find /Volumes/myshare -name .DS_Store -type f -delete
```

This is usually the cleanest. The `-type f` limits deletions to regular files.

**Method B: Using `xargs` (useful for very large trees)**

```bash
find /Volumes/myshare -name .DS_Store -type f -print0 | xargs -0 rm
```

Best practices here:
- Always **run the same `find` first with `-print` only** before adding `-delete` or piping to `rm`.
- Always use the **full path** to your share (`/Volumes/...`), never `~` or `/`, to avoid deleting outside the SMB mount.
- Consider running as your regular user instead of `sudo` unless permissions require otherwise—this reduces risk.

---

**Impact of `defaults write com.apple.desktopservices DSDontWriteNetworkStores -bool TRUE`**

This setting:

```bash
defaults write com.apple.desktopservices DSDontWriteNetworkStores -bool TRUE
killall Finder
```

(You need to restart Finder or log out/in for it to fully take effect.)

**What it does:**

- Prevents Finder from creating `.DS_Store` **on network volumes** (SMB, AFP, etc.).
- This is limited to **network** mounts; it does not affect local disks.

**What changes in practice:**

- `.DS_Store` files won’t appear on the SMB share anymore (after the change takes effect).
- Finder can’t persist folder view settings on that network share. That means:
  - Custom icon positions, view mode (icon/list/column/gallery) per-folder may not “stick” reliably on the share.
  - Sorting, column sizing, background image settings, etc., may not be remembered per folder on that share.
- On **local** drives (internal SSD, external drives attached directly), `.DS_Store` behavior is unchanged — Finder still creates them as usual.

---

### **How to Install Docker on MacBook Pro M1 Using Homebrew**

#### 1. Install Homebrew (if not already installed)

Homebrew is the starting point. If you haven’t installed it yet, follow these steps:

1. Open up your **Terminal**.
2. Paste the following command to install Homebrew:

   ```bash
   /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
   ```

3. Follow the on-screen instructions, and wait until Homebrew finishes installing.

4. Confirm Homebrew is installed by checking its version:

   ```bash
   brew --version
   ```

#### 2. Install Docker Using Homebrew Cask

After Homebrew is installed, use **Homebrew Cask** to install Docker swiftly.

1. Run the following command in the terminal:

   ```bash
   brew install --cask docker
   ```

2. Once the installation completes, Docker should now be installed. You can open **Docker** by searching for it using **Spotlight (⌘ + space)** or navigating to it in the **Applications** folder.

#### 3. Start Docker and Initial Configuration

1. Launch the **Docker application** from your Applications folder.
2. Upon first launch, Docker will handle initial setup steps, including configuration.
3. Ensure Docker is up and running by checking its status in the **menu bar** (you’ll see a Docker icon near the clock). Wait for Docker to finish starting.

---

### **Validating the Installation**

Once Docker has been installed and started, confirm that it’s installed correctly:

1. In the terminal, run:

   ```bash
   docker --version
   ```

   If Docker is running, you’ll see output similar to this (please note that M1-specific versions may vary):

   ```
   Docker version 20.10.x, build xxxx
   ```

If Docker doesn’t return a version, ensure that the Docker application is running. You can launch it from the Applications folder or use **Spotlight** to find it.

---

### **Running Your First Docker Container**

#### 1. Run the "hello-world" Docker container

Test Docker by running a simple container:

```bash
docker run hello-world
```

Upon running this command, Docker will pull the `hello-world` image from Docker Hub (if not already downloaded) and run it, outputting:

```
Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```

#### 2. Running ARM-based Containers on M1

Since your MacBook Pro M1 uses an ARM chip, you'll want to run **ARM-compatible Docker images** to maximize performance. Luckily, many popular images now support ARM, and Docker Desktop for Mac includes support for Apple Silicon.

For example, to run an **Ubuntu ARM64** image and enter an interactive bash terminal within the container, you can use the following command:

```bash
docker run -it --platform linux/arm64 ubuntu bash
```

(If you run into any issues due to architecture mismatch, Docker Desktop has the ability to use emulation via **qemu** to run x86 images, though you’ll notice a slight performance hit.)

---

### **Customizing Docker for Your Development Machine**

To make Docker even more efficient for your development environment and to tailor Docker Desktop on your M1 MacBook Pro, you can configure some preferences.

#### 1. Accessing Docker Preferences

- Open **Docker Desktop** by clicking its icon in the menu bar.
- Head to **Preferences**.

#### 2. Optimize Resource Usage

- Navigate to the **Resources** tab.
- Here, adjust how much of your system’s resources (CPU, RAM, disk storage, etc.) Docker is allowed to use.
  
Due to the high efficiency of Apple's M1 chip, Docker can typically use fewer resources compared to an Intel-based Mac, but if you're running multiple containers, adjusting these limits based on workload will benefit you.

#### 3. Automatic Startup

- In the **General** tab, toggle the option to **Start Docker Desktop when you log in**. Enable this option if you frequently use Docker for your work.

---

### **Running and Managing Containers**

Now that Docker is operational, let's explore container management:

#### Running a Web Application Inside Docker

Suppose you want to run an NGINX server locally inside a container. Forward port `8080` from your local machine into the container by running this command:

```bash
docker run -d -p 8080:80 nginx
```

This will expose the web server from the container on port **8080** of your local machine. Open `http://localhost:8080` in your browser, and you’ll see the default **NGINX Welcome page**.

#### Interacting with Running Containers

Interactive sessions inside containers provide you with a console to the container’s environment. For example, use the following command to run an interactive bash shell inside an **Ubuntu** container:

```bash
docker run -it ubuntu bash
```

This is useful for debugging, exploring, and making manual changes as needed.

---

### **Troubleshooting**

#### PATH Related Error Messages

The error occurs because Homebrew is installed successfully, but its binary path (`/opt/homebrew/bin`) is not included in your shell's `PATH` variable. As a result, your terminal can't find the `brew` command.

To fix this, you need to follow Homebrew's instructions and add the necessary command to your shell configuration file (in your case, `.zprofile`, since your system is using `zsh` as its default shell).

Here's a step-by-step guide to fix the issue:

#### Steps to resolve the error:

1. Open your terminal.

2. Run the following commands (copied from the output message) to add Homebrew to your `PATH`:

   ```bash
   echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> /Users/user/.zprofile
   eval "$(/opt/homebrew/bin/brew shellenv)"
   ```

   - The first command appends the necessary `eval` command to your `.zprofile`, ensuring that Homebrew will be sourced whenever a terminal session starts.
   
   - The second command immediately updates your `PATH` for the current terminal session, allowing you to use `brew` without restarting your terminal.

3. Verify that Homebrew is now in your `PATH` by running:

   ```bash
   brew --version
   ```

   If the installation was successful and the `PATH` is correctly updated, you should see the version of Homebrew, and you can start using `brew`.

#### Additional Notes:

- If you open a new terminal tab or session, the changes in `.zprofile` should ensure that Homebrew is automatically loaded.
  
- If you're using a different shell (e.g., `bash` or `fish`), the configuration file could be different (like `.bash_profile`, `.bashrc`, or `config.fish`). However, macOS typically uses `zsh` as the default shell, so `.zprofile` is standard for most users.

#### Checking your PATH:

To double-check that `/opt/homebrew/bin` has been added to your `PATH`, you can run:

```bash
echo $PATH
```

You should see `/opt/homebrew/bin` in the output.

Once everything is properly set, the warning should be cleared, and your Homebrew installation will work correctly.---

### **Configuration files**

- Not documented yet or not applicable for this topic.

---

### **Installation**

- To be documented.

---

### **Post-install tasks**

- To be documented.

---

### **Known issues**

- No known issues documented yet.

