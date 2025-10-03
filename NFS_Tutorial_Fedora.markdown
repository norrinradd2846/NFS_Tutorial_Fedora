# NFS Tutorial for Fedora with User and Drive Mount

This tutorial guides you through setting up an NFS server on Fedora, creating a user for NFS access, and mounting the shared drive on a client machine.

## Prerequisites
- Two Fedora systems (one for the NFS server, one for the client).
- Root or sudo access on both systems.
- Network connectivity between the server and client.
- Ensure both systems are updated: `sudo dnf update -y`.

## Step 1: Set Up the NFS Server

### 1.1 Install NFS Packages
Install the NFS server packages on the server machine.
```bash
sudo dnf install nfs-utils -y
```

### 1.2 Start and Enable NFS Services
Enable and start the necessary NFS services.
```bash
sudo systemctl enable --now nfs-server rpcbind
sudo systemctl start nfs-server rpcbind
```
Verify the services are running:
```bash
sudo systemctl status nfs-server rpcbind
```

### 1.3 Create a Directory to Share
Create a directory to share via NFS (e.g., `/srv/nfs_share`).
```bash
sudo mkdir -p /srv/nfs_share
sudo chmod 755 /srv/nfs_share
```

### 1.4 Configure NFS Exports
Edit the NFS exports file to define the shared directory.
```bash
sudo nano /etc/exports
```
Add the following line to share `/srv/nfs_share` with a client (replace `192.168.1.0/24` with your network range or specific client IP):
```
/srv/nfs_share 192.168.1.0/24(rw,sync,no_subtree_check)
```
- `rw`: Read/write access.
- `sync`: Ensures data is written to disk.
- `no_subtree_check`: Improves performance by disabling subtree checking.

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X` in nano).

### 1.5 Export the NFS Share
Apply the export configuration.
```bash
sudo exportfs -ra
```
Verify the exported directory:
```bash
sudo exportfs -v
```

### 1.6 Configure the Firewall
Allow NFS traffic through the firewall.
```bash
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --permanent --add-service=rpc-bind
sudo firewall-cmd --permanent --add-service=mountd
sudo firewall-cmd --reload
```

## Step 2: Create an NFS User

### 2.1 Create a User on the Server
Create a user (e.g., `nfsuser`) for NFS access.
```bash
sudo useradd -m nfsuser
sudo passwd nfsuser
```
Set a password when prompted.

### 2.2 Set Permissions for the Shared Directory
Ensure `nfsuser` has access to the shared directory.
```bash
sudo chown nfsuser:nfsuser /srv/nfs_share
sudo chmod 770 /srv/nfs_share
```

### 2.3 Create the Same User on the Client
To ensure consistent file ownership, create a user with the same UID and GID on the client machine.
1. Check the UID/GID of `nfsuser` on the server:
   ```bash
   id nfsuser
   ```
   Example output: `uid=1001(nfsuser) gid=1001(nfsuser)`.
2. On the client, create a user with the same UID/GID:
   ```bash
   sudo useradd -u 1001 -m nfsuser
   sudo passwd nfsuser
   ```

## Step 3: Mount the NFS Share on the Client

### 3.1 Install NFS Client Packages
On the client machine, install NFS utilities.
```bash
sudo dnf install nfs-utils -y
```

### 3.2 Create a Mount Point
Create a directory on the client to mount the NFS share.
```bash
sudo mkdir -p /mnt/nfs_share
```

### 3.3 Test the NFS Share
List available NFS shares from the server (replace `<server-ip>` with the server’s IP address).
```bash
showmount -e <server-ip>
```
You should see `/srv/nfs_share` in the output.

### 3.4 Mount the NFS Share
Mount the NFS share manually to test.
```bash
sudo mount -t nfs <server-ip>:/srv/nfs_share /mnt/nfs_share
```
Verify the mount:
```bash
df -h | grep nfs_share
```

### 3.5 Automount the NFS Share
To mount the share automatically on boot, edit `/etc/fstab`.
```bash
sudo nano /etc/fstab
```
Add the following line (replace `<server-ip>` with the server’s IP):
```
<server-ip>:/srv/nfs_share /mnt/nfs_share nfs defaults 0 0
```
Save and exit.

Test the fstab entry:
```bash
sudo mount -a
```
Verify the mount:
```bash
df -h | grep nfs_share
```

## Step 4: Test NFS Access
1. **On the client**, log in as `nfsuser`:
   ```bash
   su - nfsuser
   ```
2. Create a test file in the mounted directory:
   ```bash
   touch /mnt/nfs_share/testfile.txt
   ```
3. **On the server**, verify the file appears:
   ```bash
   ls -l /srv/nfs_share
   ```
   The file should be owned by `nfsuser`.

## Step 5: Secure the NFS Setup (Optional)
- **Restrict access**: Limit the export to specific client IPs in `/etc/exports` (e.g., `192.168.1.100(rw,sync,no_subtree_check)`).
- **Use NFSv4**: Add `fsid=0` and use `/srv/nfs_share` as the root export for better security.
  Example `/etc/exports`:
  ```
  /srv 192.168.1.0/24(rw,sync,no_subtree_check,fsid=0)
  /srv/nfs_share 192.168.1.0/24(rw,sync,no_subtree_check)
  ```
- **Enable Kerberos**: For advanced security, configure NFS with Kerberos authentication (requires additional setup).

## Troubleshooting
- **Mount fails**:
  - Ensure the server IP is correct and reachable (`ping <server-ip>`).
  - Check if NFS services are running: `sudo systemctl status nfs-server`.
  - Verify firewall settings: `sudo firewall-cmd --list-all`.
- **Permission denied**:
  - Confirm the UID/GID of `nfsuser` matches on both server and client.
  - Check directory permissions: `ls -ld /srv/nfs_share`.
- **RPC errors**:
  - Ensure `rpcbind` is running on both server and client.
- **Logs**:
  - Server logs: `sudo tail -f /var/log/messages`.
  - Client logs: `sudo tail -f /var/log/messages`.

## Notes
- NFS is not encrypted; use it in trusted networks or set up a VPN for security.
- For production, consider using NFSv4 or SSH-based alternatives like SFTP for better security.
- Replace `192.168.1.0/24` with your actual network range or client IP.

This setup provides a functional NFS server with a dedicated user and a mounted drive on the client. Let me know if you need further clarification or advanced configurations!