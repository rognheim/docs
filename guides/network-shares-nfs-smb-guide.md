---
title: How to set up a NFS share
last_reviewed: 2026-03-07
owner: Rognheim
---

# :material-nas:


## **How to set up a NFS share**

---

### **On TrueNAS Scale: Configuring the NFS Share**

**1. Access the TrueNAS Scale Web Interface**

- Open a web browser and navigate to the IP address of your TrueNAS Scale VM.
- Log in with your administrative credentials.

**2. Create a Dataset (If Not Already Created)**

- **a.** Navigate to **"Storage"** from the left-hand menu.
- **b.** Select your pool where you want to create the dataset.
- **c.** Click on the **"Add Dataset"** button.
- **d.** Provide a name for your dataset (e.g., `shared_dataset`) and configure any other desired settings.
- **e.** Click **"Save"** to create the dataset.

**3. Set Permissions on the Dataset**

- **a.** Find the dataset you just created in the list.
- **b.** Click on the **three dots** (⋮) next to the dataset name and select **"Edit Permissions"**.
- **c.** Set the **"User"** to `root` and **"Group"** to `wheel` for simplicity (you can use specific users/groups as needed).
- **d.** Ensure **"Apply permissions recursively"** is checked if you have existing files.
- **e.** Set the permission type to **POSIX ACL**.
- **f.** Click **"Save"**.

**4. Configure NFS Sharing**

- **a.** Navigate to **"Sharing"** > **"Unix Shares (NFS)"**.
- **b.** Click **"Add"** to create a new NFS share.
- **c.** In the **"Path"** field, click **"Browse"** and select the dataset you created (e.g., `/mnt/your_pool/shared_dataset`).
- **d.** Configure the **"Advanced Options"**:

  - **"Authorized Networks"**: Enter the IP address or subnet of your Raspberry Pi (e.g., `192.168.1.50/32` for a single IP).
  - **"Maproot User"**: Set to `root`.
  - **"Maproot Group"**: Set to `wheel`.
  - **Optional:** Check **"Allow Non-root Mount"** if needed.

- **e.** Click **"Save"** to create the NFS share.

**5. Start the NFS Service**

- **a.** Go to **"Services"** from the left-hand menu.
- **b.** Find **"NFS"** in the list and toggle it **"ON"**.
- **c.** Click on the **three dots** (⋮) next to **"NFS"** and select **"Start Automatically"** if you want it to start on boot.

### **On Raspberry Pi: Mounting the NFS Share**

**1. Update and Install NFS Client Utilities**

Open a terminal on your Raspberry Pi and run:

```bash
sudo apt update
sudo apt install nfs-common
```

**2. Create a Mount Point**

Choose a directory where you want the NFS share to be mounted, e.g., `/mnt/truenas_nfs`.

```bash
sudo mkdir -p /mnt/truenas_nfs
```

**3. Mount the NFS Share Manually**

Use the `mount` command to mount the NFS share:

```bash
sudo mount -t nfs -o rw,sync <TrueNAS_IP>:/mnt/your_pool/shared_dataset /mnt/truenas_nfs
```

Replace `<TrueNAS_IP>` with the IP address of your TrueNAS Scale VM (e.g., `192.168.1.100`), `your_pool` with the name of your storage pool, and `shared_dataset` with the name of your dataset.

**Example:**

```bash
sudo mount -t nfs -o rw,sync 192.168.1.100:/mnt/tank/shared_dataset /mnt/truenas_nfs
```

**4. Verify the Mount**

Check if the NFS share is mounted correctly:

```bash
df -h | grep truenas_nfs
```

List the contents:

```bash
ls -la /mnt/truenas_nfs
```

**5. Test Read/Write Access**

Try creating a test file:

```bash
sudo touch /mnt/truenas_nfs/testfile
```

If you encounter permission issues, you may need to adjust ownership or permissions.

**6. Configure Automatic Mounting**

To mount the NFS share automatically at boot, you can add an entry to `/etc/fstab`.

**a. Open `/etc/fstab` with a text editor:**

```bash
sudo nano /etc/fstab
```

**b. Add the following line at the end of the file:**

```
<TrueNAS_IP>:/mnt/your_pool/shared_dataset /mnt/truenas_nfs nfs defaults,rw,sync 0 0
```

**Example:**

```
192.168.1.100:/mnt/tank/shared_dataset /mnt/truenas_nfs nfs defaults,rw,sync 0 0
```

**c. Save and exit the editor (Ctrl+O, Enter, Ctrl+X in nano).**

**d. Test the fstab entry:**

First, unmount the share:

```bash
sudo umount /mnt/truenas_nfs
```

Then, mount all filesystems from fstab:

```bash
sudo mount -a
```

Check if the share is mounted:

```bash
df -h | grep truenas_nfs
```

### **Troubleshooting Tips**

- **Permission Issues:**

  If you cannot read or write to the NFS share due to permissions, you may need to adjust the permissions on TrueNAS:

  - Ensure that **"Maproot User"** and **"Maproot Group"** are set to `root` in the NFS share settings.
  - Alternatively, map the NFS share to a specific user that exists on both systems with matching UID and GID.

- **Firewall Settings:**

  Ensure that any firewalls between the Raspberry Pi and TrueNAS are configured to allow NFS traffic (typically on TCP and UDP port 2049).

- **NFS Version Compatibility:**

  By default, NFSv4 is used. If you need to specify the NFS version, you can add the `nfsvers` option:

  ```bash
  sudo mount -t nfs -o rw,sync,nfsvers=4 <TrueNAS_IP>:/mnt/your_pool/shared_dataset /mnt/truenas_nfs
  ```

- **Checking Exported NFS Shares:**

  To see available NFS shares from TrueNAS on the Raspberry Pi, run:

  ```bash
  showmount -e <TrueNAS_IP>
  ```

### **Additional Notes**

- **Security Considerations:**

  NFS does not provide encryption by default. For secure environments, consider using a VPN or network isolation.

- **User and Group IDs:**

  NFS relies on matching user and group IDs between client and server. Ensure that the user IDs (UID) and group IDs (GID) for the users accessing the share match on both the Raspberry Pi and the TrueNAS server.

- **Alternative: Using Autofs for On-Demand Mounting**

  Instead of mounting at boot, you can use `autofs` to mount the NFS share on demand. This can prevent boot delays if the network share is unavailable.

  **Install Autofs:**

  ```bash
  sudo apt install autofs
  ```

  **Configure Autofs:**

  - Edit the master map file:

    ```bash
    sudo nano /etc/auto.master
    ```

    Add the following line:

    ```
    /- /etc/auto.nfs
    ```

  - Create the map file:

    ```bash
    sudo nano /etc/auto.nfs
    ```

    Add the following line:

    ```
    /mnt/truenas_nfs -fstype=nfs,rw,sync <TrueNAS_IP>:/mnt/your_pool/shared_dataset
    ```

  - Restart the Autofs service:

    ```bash
    sudo systemctl restart autofs
    ```

## **How to set up a SMB share**

---

### **On TrueNAS Scale: Configuring the SMB Share**

**1. Access the TrueNAS Scale Web Interface**

- Open a web browser and navigate to the IP address of your TrueNAS Scale VM.
- Log in with your administrative credentials.

**2. Create a Dataset (If Not Already Created)**

- **a.** Navigate to **"Storage"** from the left-hand menu.
- **b.** Select your pool where you want to create the dataset.
- **c.** Click on the **"Add Dataset"** button.
- **d.** Provide a name for your dataset (e.g., `shared_dataset`) and configure any other desired settings.
- **e.** Click **"Save"** to create the dataset.

**3. Set Permissions on the Dataset**

- **a.** Find the dataset you just created in the list.
- **b.** Click on the **three dots** (⋮) next to the dataset name and select **"Edit Permissions"**.
- **c.** Set the **"User"** and **"Group"** to appropriate values. For simplicity, you can use the user `nobody` and group `nogroup` for guest access, or create a specific user and group for authenticated access.
- **d.** Ensure that the permissions allow the desired level of access (read/write).
- **e.** Click **"Save"**.

**4. Configure SMB Sharing**

- **a.** Navigate to **"Sharing"** > **"Windows Shares (SMB)"**.
- **b.** Click **"Add"** to create a new SMB share.
- **c.** In the **"Path"** field, click **"Browse"** and select the dataset you created (e.g., `/mnt/your_pool/shared_dataset`).
- **d.** Provide a **"Name"** for your SMB share (this will be the share name you see on the network).
- **e.** Configure additional settings:

  - **"Allow Guest Access"**: Enable this if you want to allow access without authentication.
  - **"Enable ACL"**: Ensure this is enabled to manage permissions.
  - **"Purpose"**: You can select **"No presets"** or choose one that fits your use case.

- **f.** Click **"Save"** to create the SMB share.

**5. Start the SMB Service**

- **a.** Go to **"Services"** from the left-hand menu.
- **b.** Find **"SMB"** in the list and toggle it **"ON"**.
- **c.** Click on the **three dots** (⋮) next to **"SMB"** and select **"Start Automatically"** if you want it to start on boot.

**6. (Optional) Configure SMB Version Settings**

- **a.** Navigate to **"Services"** > **"SMB"** and click on the **"Configure"** button.
- **b.** In the SMB service settings, you can set the **"Minimum SMB Protocol"** and **"Maximum SMB Protocol"** if needed.

  - By default, TrueNAS uses SMB 2 and SMB 3 protocols.
  - If you encounter compatibility issues, you can adjust these settings.

- **c.** Click **"Save"** if you made any changes.

### **On Raspberry Pi: Mounting the SMB Share**

**1. Update and Install CIFS Utilities**

Open a terminal on your Raspberry Pi and run:

```bash
sudo apt update
sudo apt install cifs-utils
```

**2. Create a Mount Point**

Choose a directory where you want the SMB share to be mounted, e.g., `/mnt/truenas_smb`.

```bash
sudo mkdir -p /mnt/truenas_smb
```

**3. Mount the SMB Share Manually**

**a. Mounting as Guest (No Authentication)**

If you allowed guest access on the SMB share:

```bash
sudo mount -t cifs -o guest,uid=pi,gid=pi //TrueNAS_IP/share_name /mnt/truenas_smb
```

- Replace `TrueNAS_IP` with the IP address of your TrueNAS Scale VM (e.g., `192.168.1.100`).
- Replace `share_name` with the name of your SMB share.
- `uid=pi` and `gid=pi` set the ownership of the mounted files to your Raspberry Pi user (`pi` is the default user; adjust if different).

**Example:**

```bash
sudo mount -t cifs -o guest,uid=pi,gid=pi //192.168.1.100/shared_dataset /mnt/truenas_smb
```

**b. Mounting with Username and Password**

If you require authentication:

```bash
sudo mount -t cifs -o username=your_username,password=your_password,uid=pi,gid=pi //TrueNAS_IP/share_name /mnt/truenas_smb
```

Replace `your_username` and `your_password` with your SMB user credentials.

**4. Verify the Mount**

Check if the SMB share is mounted correctly:

```bash
df -h | grep truenas_smb
```

List the contents:

```bash
ls -la /mnt/truenas_smb
```

**5. Test Read/Write Access**

Try creating a test file:

```bash
touch /mnt/truenas_smb/testfile
```

If you encounter permission issues, you may need to adjust ownership or permissions on the TrueNAS side.

**6. Configure Automatic Mounting**

To mount the SMB share automatically at boot, you can add an entry to `/etc/fstab`.

**a. Create a Credentials File (Optional but Recommended)**

For security, store your credentials in a separate file.

- Create a file (e.g., `/home/pi/.smbcredentials`):

  ```bash
  nano /home/pi/.smbcredentials
  ```

- Add the following lines:

  ```
  username=your_username
  password=your_password
  ```

- Save and exit.

- Secure the credentials file:

  ```bash
  chmod 600 /home/pi/.smbcredentials
  ```

**b. Edit `/etc/fstab`**

Open `/etc/fstab`:

```bash
sudo nano /etc/fstab
```

Add the following line at the end of the file:

- If using credentials file:

  ```
//TrueNAS_IP/share_name /mnt/truenas_smb cifs credentials=/home/pi/.smbcredentials,uid=pi,gid=pi,iocharset=utf8,nofail 0 0
  ```

- If mounting as guest:

  ```
//TrueNAS_IP/share_name /mnt/truenas_smb cifs guest,uid=pi,gid=pi,iocharset=utf8,nofail 0 0
  ```

**Example:**

```
//192.168.1.100/shared_dataset /mnt/truenas_smb cifs guest,uid=pi,gid=pi,iocharset=utf8,nofail 0 0
```

- `nofail`: Allows the boot process to continue even if the network share is not available.
- `iocharset=utf8`: Ensures proper encoding of file names.
- `uid` and `gid`: Set ownership to your Raspberry Pi user.

**c. Save and exit the editor (Ctrl+O, Enter, Ctrl+X in nano).**

**d. Test the fstab entry:**

First, unmount the share if it's already mounted:

```bash
sudo umount /mnt/truenas_smb
```

Then, mount all filesystems from fstab:

```bash
sudo mount -a
```

Check if the share is mounted:

```bash
df -h | grep truenas_smb
```

**7. (Optional) Specify SMB Protocol Version**

If you encounter issues with the SMB protocol version, you can specify it in the mount options.

For example, to force SMB version 3.0:

```bash
sudo mount -t cifs -o vers=3.0,guest,uid=pi,gid=pi //192.168.1.100/shared_dataset /mnt/truenas_smb
```

In `/etc/fstab`, add `,vers=3.0` to the options:

```
//192.168.1.100/shared_dataset /mnt/truenas_smb cifs guest,uid=pi,gid=pi,vers=3.0,iocharset=utf8,nofail 0 0
```

**Note:** The `vers` option specifies the SMB protocol version. Supported versions are `1.0`, `2.0`, `2.1`, `3.0`, `3.1.1`. SMBv1 is outdated and not secure; it's recommended to use SMBv2 or higher.

### **Troubleshooting Tips**

- **Permission Issues:**

  - Ensure that the users and groups have appropriate permissions on the TrueNAS side.
  - You may need to adjust the `uid` and `gid` options to match the user on the Raspberry Pi.
  - On TrueNAS, check that the dataset permissions and SMB share settings allow access.

- **Firewall Settings:**

  - Make sure any firewalls on TrueNAS or Raspberry Pi allow SMB traffic (TCP ports 139 and 445).

- **SMB Version Compatibility:**

  - If you receive errors like "mount error(95): Operation not supported," try specifying different SMB protocol versions with the `vers` option.
  - Example errors and solutions:

    - **Error:** `mount error(112): Host is down`
      - **Solution:** Verify the TrueNAS IP and that the SMB service is running.

    - **Error:** `mount error(13): Permission denied`
      - **Solution:** Check credentials and permissions on the TrueNAS share.

- **Logs and Diagnostics:**

  - View system logs for error messages:

    ```bash
    dmesg | tail
    ```

  - Test the SMB connection using `smbclient`:

    ```bash
    smbclient -L //TrueNAS_IP/ -U your_username
    ```

- **Ensuring Network Connectivity:**

  - Confirm that the Raspberry Pi can reach the TrueNAS server:

    ```bash
    ping 192.168.1.100
    ```

### **Additional Notes**

- **Security Considerations:**

  - Storing passwords in plain text (even in a credentials file) can be a security risk if unauthorized users can access the file.
  - Ensure that the `.smbcredentials` file has correct permissions (`chmod 600`).

- **User and Group IDs:**

  - The `uid` and `gid` options in the mount command set the ownership of the mounted files to the specified user and group on the Raspberry Pi.
  - Run `id` on Raspberry Pi to find your user and group IDs.

    ```bash
    id
    ```

- **File Permissions:**

  - If you need to set specific file and directory permissions, add `file_mode` and `dir_mode` options:

    ```bash
    sudo mount -t cifs -o guest,uid=pi,gid=pi,file_mode=0666,dir_mode=0777 //TrueNAS_IP/share_name /mnt/truenas_smb
    ```

- **Using Autofs for On-Demand Mounting**

  - Autofs can automatically mount the SMB share when it's accessed, which can prevent boot delays if the network share is unavailable.

  **Install Autofs:**

  ```bash
  sudo apt install autofs
  ```

  **Configure Autofs:**

  - Edit the master map file:

    ```bash
    sudo nano /etc/auto.master
    ```

    Add the following line:

    ```
    /- /etc/auto.cifs
    ```

  - Create the map file:

    ```bash
    sudo nano /etc/auto.cifs
    ```

    Add the following line:

    ```
    /mnt/truenas_smb -fstype=cifs,credentials=/home/pi/.smbcredentials,uid=pi,gid=pi,iocharset=utf8 ://TrueNAS_IP/share_name
    ```

  - Restart the Autofs service:

    ```bash
    sudo systemctl restart autofs
    ```

- **Performance Tuning:**

  - You can improve performance by adjusting mount options. For example, adding `cache=none` or `cache=strict`.

    ```bash
    sudo mount -t cifs -o guest,uid=pi,gid=pi,cache=none //192.168.1.100/shared_dataset /mnt/truenas_smb
    ```

- **Alternative Access Methods:**

  - **FTP or SFTP:** If SMB is not working as desired, consider setting up FTP or SFTP services on TrueNAS.
  - **NFS:** As previously discussed, NFS is also a good option, especially in Unix/Linux environments.

---

