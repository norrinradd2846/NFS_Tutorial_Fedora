# NFS Tutorial: Restrict Access and Login to Only nfsuser on Fedora

This tutorial modifies an existing NFS server setup on Fedora to ensure only the `nfsuser` can access and log in to the NFS share. It assumes you have already set up an NFS server and created the `nfsuser` as described in the previous tutorial.

## Prerequisites
- NFS server installed and running (`nfs-utils`).
- `nfsuser` created on both server and client with matching UID/GID.
- Shared directory (e.g., `/srv/nfs_share`) configured.
- Network connectivity between server and client.

## Step 1: Restrict NFS Access to nfsuser

### 1.1 Modify NFS Exports
Edit the `/etc/exports` file to restrict access to the NFS share.
```bash
sudo nano /etc/exports
```
Update the export to include the `no_all_squash` and `anonuid/anongid` options to map all access to `nfsuser`. Replace `192.168.1.0/24` with your network range or client IP:
```
/srv/nfs_share 192.168.1.0/24(rw,sync,no_subtree_check,no_all_squash,anonuid=1001,anongid=1001)
```
- `no_all_squash`: Ensures user IDs are not mapped to the anonymous user.
- `anonuid=1001,anongid=1001`: Maps all NFS access to the UID/GID of `nfsuser` (replace `1001` with the actual UID/GID of `nfsuser`).

Verify the UID/GID of `nfsuser`:
```bash
id nfsuser
```
Example output: `uid=1001(nfsuser) gid=1001(nfsuser)`.

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X` in nano).

### 1.2 Re-export the NFS Configuration
Apply the updated exports:
```bash
sudo exportfs -ra
```
Verify the export:
```bash
sudo exportfs -v
```

### 1.3 Set Directory Permissions
Ensure only `nfsuser` has access to the shared directory.
```bash
sudo chown nfsuser:nfsuser /srv/nfs_share
sudo chmod 700 /srv/nfs_share
```
- `chown`: Sets ownership to `nfsuser`.
- `chmod 700`: Restricts access to only the owner (`nfsuser`).

## Step 2: Secure NFS Server Login

### 2.1 Disable Other User Logins
To prevent other users from logging into the server via SSH or other methods, configure the system to allow only `nfsuser` (and optionally `root` for administration).

#### Option 1: Restrict SSH Logins
Edit the SSH configuration file `/etc/ssh/sshd_config`:
```bash
sudo nano /etc/ssh/sshd_config
```
Add or modify the following line to allow only `nfsuser` (and `root` if needed):
```
AllowUsers nfsuser root
```
Save and exit.

Restart the SSH service:
```bash
sudo systemctl restart sshd
```

#### Option 2: Use PAM to Restrict Logins
Restrict all logins to `nfsuser` using PAM (Pluggable Authentication Modules).
1. Edit `/etc/security/access.conf`:
   ```bash
   sudo nano /etc/security/access.conf
   ```
   Add the following lines at the end:
   ```
   + : nfsuser : ALL
   + : root : ALL
   - : ALL : ALL
   ```
   - This allows `nfsuser` and `root` to log in from any source and denies all other users.

2. Configure PAM to use `access.conf`:
   Edit `/etc/pam.d/sshd`:
   ```bash
   sudo nano /etc/pam.d/sshd
   ```
   Add the following line at the top:
   ```
   account required pam_access.so
   ```
   Save and exit.

3. Restart the SSH service:
   ```bash
   sudo systemctl restart sshd
   ```

### 2.2 Lock Other User Accounts
To prevent other users from logging in, lock their accounts (except `nfsuser` and `root`).
List all users:
```bash
getent passwd
```
Lock accounts for non-essential users (e.g., `otheruser`):
```bash
sudo passwd -l otheruser
```
This sets an invalid password, preventing login.

## Step 3: Verify NFS Access on the Client

### 3.1 Ensure Matching nfsuser on Client
On the client, ensure `nfsuser` exists with the same UID/GID as on the server:
```bash
id nfsuser
```
If needed, create or modify `nfsuser`:
```bash
sudo useradd -u 1001 -m nfsuser
sudo passwd nfsuser
```

### 3.2 Mount the NFS Share
Mount the NFS share on the client (replace `<server-ip>` with the server’s IP):
```bash
sudo mount -t nfs <server-ip>:/srv/nfs_share /mnt/nfs_share
```
Verify the mount:
```bash
df -h | grep nfs_share
```

### 3.3 Test Access as nfsuser
Switch to `nfsuser` on the client:
```bash
su - nfsuser
```
Create a file in the mounted directory:
```bash
touch /mnt/nfs_share/testfile.txt
```
Verify the file on the server:
```bash
ls -l /srv/nfs_share
```
The file should be owned by `nfsuser`.

### 3.4 Test Access as Another User
Switch to a different user on the client (e.g., `otheruser`):
```bash
su - otheruser
```
Attempt to access the NFS share:
```bash
touch /mnt/nfs_share/testfile2.txt
```
This should fail with a "Permission denied" error due to the `anonuid/anongid` mapping and directory permissions.

## Step 4: Configure Firewall
Ensure only NFS-related traffic is allowed.
```bash
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --permanent --add-service=rpc-bind
sudo firewall-cmd --permanent --add-service=mountd
sudo firewall-cmd --reload
```
Optionally, restrict to a specific client IP:
```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.100" service name="nfs" accept'
sudo firewall-cmd --reload
```

## Step 5: Troubleshooting
- **Permission denied**:
  - Verify `anonuid` and `anongid` match `nfsuser`’s UID/GID in `/etc/exports`.
  - Check directory permissions: `ls -ld /srv/nfs_share`.
- **NFS mount fails**:
  - Ensure the server IP is correct and reachable: `ping <server-ip>`.
  - Check NFS services: `sudo systemctl status nfs-server`.
- **Login issues**:
  - Verify `AllowUsers` in `/etc/ssh/sshd_config` or PAM settings in `/etc/security/access.conf`.
  - Check user lock status: `sudo passwd -S otheruser`.
- **Logs**:
  - Server: `sudo tail -f /var/log/messages`.
  - Client: `sudo tail -f /var/log/messages`.

## Notes
- This setup maps all NFS access to `nfsuser` via `anonuid/anongid`, ensuring only `nfsuser`’s credentials are used.
- For production, consider NFSv4 with Kerberos for stronger authentication or use SSH-based solutions like SFTP.
- Restricting logins to `nfsuser` may limit administrative access; keep `root` or another admin user for management.

This configuration ensures only `nfsuser` can access and interact with the NFS share, with login restrictions enforced on the server.