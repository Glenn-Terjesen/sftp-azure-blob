# Description
For now, a howto setup a secure sftp service with azure blob storage.

# Requirements
An Azure account with a active subscrition.

# Create VM
Login to Azure, create Ubuntu 18.04 VM, allow internet access to port 22/tcp (default)
When it's created, add a data disk to handle bigger uploads of files. ie 50GB Standard HDD.

# Create a Azure Storage Account
Search up storage account -> new, same resource group, remember the name.
When it's created go to the resource and create a blob container, remember the name.
Go to "Access keys" under the storage account, note one of the account keys.

# VM config
Login to the vm with ssh and the key provided.
```
sudo su - (not best practice.. but lets do this..)
mkdir /azure/setup
mkdir /azure/blob
mkdir /azure/cachedisk
chmod 755 /azure -R
```
## New partition
create partition with fdisk from the newly added disk (/dev/sdc?) - I'll not go into details.. 
mkfs.ext4 on the new partition, again I'll not go into details.
add mountpoint to /etc/fstab /azure/cachedisk, get the UUID from the blkid command
```PARTUUID="e5c2fs75-01" /azure/cachedisk ext4 defaults 0 0```

# Azure Blob storage mounting
Let's mount the Azure Blob storage
```
wget https://packages.microsoft.com/config/ubuntu/18.04/packages-microsoft-prod.deb
dpkg -i packages-microsoft-prod.deb
apt-get update
apt-get install blobfuse
cd /azure/setup
touch fuse_connection.cfg
nano mountblob.sh
```
Insert the following contents
```
#!/bin/sh
if [ ! -d "/mnt/resource/blobfusetmp" ]
then
    mkdir /mnt/resource/blobfusetmp -p
fi
/usr/bin/blobfuse /azure/blob --tmp-path=/azure/cachedisk --config-file=/azure/setup/fuse_connection.cfg -o attr_timeout=240 -o entry_timeout=240 -o negative_timeout=120 -o umask=0022 -o allow_other
# umask 0022 = 755 (reversed 0777 - 0022 = 0755)
```
Make it executeable
```
chmod 700 mountblob.sh
```
*BigNote*, the mount path needs to be a subfolder, not /azureblob, due to openssh will check the parent folders filepermissions using ChrootDirectory.
Let's insert the blob acccount info
```
nano /azure/setup/fuse_connection.cfg
```
insert the following, replace the values with the Azure storage account/blob container info
```
accountName AzureStorageAccountName
accountKey /Cr2+/s7Z_Something_strange_key
containerName sftpdata01
```
let's check if it works, run
```
/azure/setup/mountblob.sh
```
you should not receive any output

Check mounts: 
```
df -h
```
you should see blobfuse /azureblob in the list

Mount at boot, edit fstab
```
nano /etc/fstab
```
add line:
```
/azure/setup/mountblob.sh /azure/blob fuse defaults,_netdev
```

Test it
```
nano /azure/blob/test.txt
```
add the contents "hello world" and save.
You should see it in the azure blob.

# SFTP service setup
```
#Add a group that the users are a member of
groupadd sftp_users
 
# edit openssh server
nano /etc/ssh/sshd_config
 
#find the line with PermitRootLogin
#uncomment/change it to
PermitRootLogin no
 
#find the line with ClientAliveInterval
#unncomment/change it to: (disconnects clients after a while)
ClientAliveInterval 5
 
# add the following 4 lines at the bottom of the file
Match Group sftp_users
      ChrootDirectory /azure/blob/%u
      ForceCommand internal-sftp
      AllowTcpForwarding no
       
 
# save and restart sshd
systemctl restart sshd
```

## User setup
create this file in /azure/setup, name it addsftpuser.sh
```
#!/bin/bash
if [ $(id -u) -eq 0 ]; then
        read -p "Enter username : " username
        read -s -p "Enter password : " password
        egrep "^$username" /etc/passwd >/dev/null
        if [ $? -eq 0 ]; then
                echo ""
                echo "ERROR: $username exists!"
                exit 1
        else
                useradd -g sftp_users -d / -s /sbin/nologin "$username"
                echo -e "$password\n$password" | passwd "$username"
                mkdir -p /azure/blob/$username
                [ $? -eq 0 ] && echo "User has been added to system!" || echo "Failed to add a user!"
        fi
else
        echo "Only root may add a user to the system."
        exit 2
fi
```
Run it and add a test account.

Test the newly created account with ie Winscp or Filezilla.

# Hardening
* Install denyhost to prevent brute-force
** apt install denyhosts
** Easy to configure: https://anansewaa.com/how-to-install-and-configure-denyhosts-on-ubuntu-18-04-lts/
* Use keyfiles for authentication or both keyfiles and password.
* Use a long password
