# EBS-Shrinker

The following script is provided to demonstrate how to resize an ebs root volume. The end-to-end solution will launch a demo EC2 instance with an initial 50Gb ebs gp2 root volume, and then resize it to 20Gb. 

**Note**: Please install the [jq utility](https://stedolan.github.io/jq/) before continuing - Steps 2 & 4 below are dependent on it.

**Note**: This script is provided to demonstrate how to perform root volume resizing - take extreme care when performing the same actions within a production environment. For extra safety measures - always snapshot any ebs volume BEFORE performing any resizing actions on it.

# Example:

[![asciicast](https://asciinema.org/a/307319.svg)](https://asciinema.org/a/307319)

## Before Resizing
Pause at 1:36 (original root volume 50Gb)

```
Filesystem      Size  Used Avail Use% Mounted on                                                                               
udev            481M     0  481M   0% /dev                                                                                     
tmpfs            99M  748K   98M   1% /run                                                                                     
/dev/xvda1       49G  1.1G   48G   3% /                                                                                        
tmpfs           492M     0  492M   0% /dev/shm                                                                                 
tmpfs           5.0M     0  5.0M   0% /run/lock                                                                                
tmpfs           492M     0  492M   0% /sys/fs/cgroup                                                                           
/dev/loop0       18M   18M     0 100% /snap/amazon-ssm-agent/1480                                                              
/dev/loop1       90M   90M     0 100% /snap/core/8268                                                                          
tmpfs            99M     0   99M   0% /run/user/1000 
```

## After Resizing
Pause at 3:52 (resized root volume 20Gb)

```
Filesystem      Size  Used Avail Use% Mounted on                                                                               
udev            481M     0  481M   0% /dev                                                                                     
tmpfs            99M  740K   98M   1% /run                                                                                     
/dev/xvda        20G  1.3G   18G   7% /                                                                                        
tmpfs           492M     0  492M   0% /dev/shm                                                                                 
tmpfs           5.0M     0  5.0M   0% /run/lock                                                                                
tmpfs           492M     0  492M   0% /sys/fs/cgroup                                                                           
/dev/loop0       18M   18M     0 100% /snap/amazon-ssm-agent/1480                                                              
/dev/loop1       90M   90M     0 100% /snap/core/8268                                                                          
tmpfs            99M     0   99M   0% /run/user/1000     
```

# Script

## Environment Variables 1

Defaults to configure the deployment of a demo EC2 instance into the Oregon (```REGION```) region. The EC2 instance will be configured with the Ubuntu 18.04 OS (```IMAGE_ID```) installed on a 50Gb (```VOLUME_SIZE_LARGE```) sized root volume - which will then be cloned and downsized to a 20Gb (```VOLUME_SIZE_SMALL```) root volume

```
REGION=us-west-2
AZ=us-west-2c
IMAGE_ID=ami-0d1cd67c26f5fca19 #ubuntu 18.04
USER=ubuntu
VOLUME_SIZE_LARGE=50 #starting size for ebs root volume
VOLUME_SIZE_SMALL=20 #final size after resizing for ebs root volume
```

## Environment Variables 2

You need to update the following environment variables which will be used later to establish a demo EC2 instance with an initial root volume of 50Gb. The demo EC2 instance should be deployed into a subnet with a security group that ensures internet connectivity both in and out.  

```
SSH_KEY_NAME=<ec2 ssh keyname here - example: DemoSSHKey>
SSH_KEY_PATH=<ec2 ssh key path here - example: ~/.ssh/DemoSSHKey.pem>
SUBNET_ID=<subnet id here - where demo ec2 instance is deployed - example: subnet-aa598002>
SECURITY_GROUP_ID=<security group id here - needs to allow inbound ssh connections to the demo ec2 instance - example: >
```

## Step 1

Launch a demo EC2 instance with a 50Gb root volume with Ubuntu 18.04 installed

```
CREATE_INSTANCE_RESULT=`aws ec2 run-instances \
    --region $REGION \
    --image-id $IMAGE_ID\
    --count 1 \
    --instance-type t2.micro \
    --key-name $SSH_KEY_NAME \
    --security-group-ids $SECURITY_GROUP_ID \
    --subnet-id $SUBNET_ID \
    --associate-public-ip-address \
    --block-device-mapping "[ { \"DeviceName\": \"/dev/sda1\", \"Ebs\": { \"VolumeSize\": $VOLUME_SIZE_LARGE, \"VolumeType\": \"gp2\" } } ]" \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=CloudAcademy.EBS.Resizer.Demo}]" "ResourceType=volume,Tags=[{Key=Name,Value=EBSLargeVol}]"`
```

Outcome:

![step 1](/doc/step1.png)

## Step 2

Retrieve the demo EC2 instance id just launched.

**Note**: Please install the [jq utility](https://stedolan.github.io/jq/) before continuing - this step is dependent on it.

```
INSTANCE_ID=`echo $CREATE_INSTANCE_RESULT | jq -r .Instances[0].InstanceId`

echo INSTANCE_ID=$INSTANCE_ID
```

## Step 3

Query and display all volumes currently attached to the demo EC2 instance. At the moment there should be just the one 50Gb GP2 root volume attached.

```
aws ec2 describe-volumes \
    --region $REGION \
    --filters Name=attachment.instance-id,Values=$INSTANCE_ID
```

## Step 4

Create a smaller 20Gb (```VOLUME_SIZE_SMALL```) gp2 volume in the same availabilty zone. Capture the volume id and echo it back out to the console.

**Note**: Please install the [jq utility](https://stedolan.github.io/jq/) before continuing - this step is dependent on it.

```
CREATE_VOL_RESULT=`aws ec2 create-volume \
    --region $REGION \
    --volume-type gp2 \
    --size $VOLUME_SIZE_SMALL \
    --availability-zone $AZ \
    --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=EBSSmallVol}]'`

NEW_VOL_ID=`echo $CREATE_VOL_RESULT | jq -r .VolumeId`

echo NEW_VOL_ID=$NEW_VOL_ID
```

Outcome:

![step 4](/doc/step4.png)

## Step 5

Stop the demo EC2 instance in preparation for attaching the new smaller volume.

```
aws ec2 stop-instances  \
    --region $REGION \
    --instance-ids $INSTANCE_ID

aws ec2 wait instance-stopped \
    --region $REGION \
    --instance-ids $INSTANCE_ID
```

Outcome:

![step 5](/doc/step5.png)

## Step 6

Attach the new smaller 20Gb volume to the EC2 demo instance and then restart it. The new volume will be attached to the ```/dev/xvdf``` device. 

```
aws ec2 attach-volume \
    --region $REGION \
    --volume-id $NEW_VOL_ID \
    --instance-id $INSTANCE_ID \
    --device /dev/xvdf   

aws ec2 start-instances  \
    --region $REGION \
    --instance-ids $INSTANCE_ID

aws ec2 wait instance-running \
    --region $REGION \
    --instance-ids $INSTANCE_ID
```

Outcome:

![step 6](/doc/step6.png)

## Step 7

Retrieve the auto assigned public IP address and then SSH into the demo EC2 instance and run the commands ```lsblk``` and ```df -h``` to examine and display the attached devices.

```
PUBLIC_IP=`aws ec2 describe-instances \
    --region $REGION \
    --filters \
    "Name=instance-state-name,Values=running" \
    "Name=instance-id,Values=$INSTANCE_ID" \
    --query 'Reservations[*].Instances[*].[PublicIpAddress]' \
    --output text`

echo PUBLIC_IP=$PUBLIC_IP

ssh -o "StrictHostKeyChecking no" -v -i $SSH_KEY_PATH $USER@$PUBLIC_IP 'bash -s' <<\EOF
lsblk
echo
df -h
EOF
```

Output will be something like:

```
Filesystem      Size  Used Avail Use% Mounted on                                                                               
udev            481M     0  481M   0% /dev                                                                                     
tmpfs            99M  748K   98M   1% /run                                                                                     
/dev/xvda1       49G  1.1G   48G   3% /                                                                                        
tmpfs           492M     0  492M   0% /dev/shm                                                                                 
tmpfs           5.0M     0  5.0M   0% /run/lock                                                                                
tmpfs           492M     0  492M   0% /sys/fs/cgroup                                                                           
/dev/loop0       18M   18M     0 100% /snap/amazon-ssm-agent/1480                                                              
/dev/loop1       90M   90M     0 100% /snap/core/8268                                                                          
tmpfs            99M     0   99M   0% /run/user/1000 
```

**Note**: The root volume is 50Gb

## Step 8

:muscle::muscle::muscle:

This is where the heavy lifting (resizing) happens!!

The resizing process involves the following sub steps:

* Format the new ```/dev/xvdf``` volume using ```mkfs -t ext4```
* Create a new directory ```/mnt/new-root-volume``` as the mount point for the ```/dev/xvdf``` device
* Rsync all root volume files/folders recursively to the ```/mnt/new-root-volume``` directory
* Install grub (boot loader) on the new ```/mnt/new-root-volume``` directory
* Unmount the new ```/mnt/new-root-volume``` directory
* Copy across both the UUID and Label from the original root volume to the new root volume

```
ssh -o "StrictHostKeyChecking no" -v -i $SSH_KEY_PATH $USER@$PUBLIC_IP 'bash -s' <<\EOF
echo shrink root vol start...
lsblk
NEW_ROOT_DIR=/mnt/new-root-volume
NEW_DEV=/dev/xvdf
sudo file -s $NEW_DEV
sudo mkfs -t ext4 $NEW_DEV
sudo mkdir $NEW_ROOT_DIR
sudo mount $NEW_DEV $NEW_ROOT_DIR
df -h
echo rsyncing start...
sudo rsync -axv / $NEW_ROOT_DIR/
echo install grub...
sudo grub-install --root-directory=$NEW_ROOT_DIR --force $NEW_DEV
sudo umount $NEW_ROOT_DIR
echo copy uuid across to new root dev...
ROOT_DEV=`sudo blkid -L cloudimg-rootfs`
UUID=`blkid -s UUID -o value $ROOT_DEV`
sudo tune2fs -U $UUID $NEW_DEV
echo copy system label across to new root vol...
LABEL=`sudo e2label $ROOT_DEV`
sudo e2label $NEW_DEV $LABEL
echo shrink root vol finished!!
EOF
```

Outcome:

![step 8](/doc/step8.png)

## Step 9

Shutdown the demo EC2 instance in preparation for detaching the EBS volumes.

```
aws ec2 stop-instances  \
    --region $REGION \
    --instance-ids $INSTANCE_ID

aws ec2 wait instance-stopped \
    --region $REGION \
    --instance-ids $INSTANCE_ID
```

Outcome:

![step 9](/doc/step9.png)

## Step 10

Detach both EBS volumes, and then reattach the smaller and newer 20Gb ebs gp2 root volume and map it to the root device at ```/dev/sda1```.

```
for vol in `aws ec2 describe-volumes \
                --region $REGION \
                --filters Name=attachment.instance-id,Values=$INSTANCE_ID \
                --query 'Volumes[*].Attachments[*].VolumeId' --output text`; \
    do aws --region us-west-2 ec2 detach-volume --volume-id $vol; \
done

aws ec2 attach-volume \
    --region $REGION \
    --volume-id $NEW_VOL_ID \
    --instance-id $INSTANCE_ID \
    --device /dev/sda1
```

Outcome:

![step 10](/doc/step10.png)

## Step 11

Restart the demo EC2 instance and query and display the updated auto assigned public IP address 

```
aws ec2 start-instances  \
    --region $REGION \
    --instance-ids $INSTANCE_ID

aws ec2 wait instance-running \
    --region $REGION \
    --instance-ids $INSTANCE_ID

PUBLIC_IP=`aws ec2 describe-instances \
    --region $REGION \
    --filters \
    "Name=instance-state-name,Values=running" \
    "Name=instance-id,Values=$INSTANCE_ID" \
    --query 'Reservations[*].Instances[*].[PublicIpAddress]' \
    --output text`

echo PUBLIC_IP=$PUBLIC_IP
```

Outcome:

![step 11](/doc/step11.png)

## Step 12

 SSH into the demo EC2 instance and run the commands ```lsblk``` and ```df -h``` to examine and display the attached config of the root device. Compare with the previous values (see previous Step 7). 

```
ssh -o "StrictHostKeyChecking no" -v -i $SSH_KEY_PATH $USER@$PUBLIC_IP 'bash -s' <<\EOF
lsblk
echo
df -h
EOF
```

Output will be something like:

```
Filesystem      Size  Used Avail Use% Mounted on                                                                               
udev            481M     0  481M   0% /dev                                                                                     
tmpfs            99M  740K   98M   1% /run                                                                                     
/dev/xvda        20G  1.3G   18G   7% /                                                                                        
tmpfs           492M     0  492M   0% /dev/shm                                                                                 
tmpfs           5.0M     0  5.0M   0% /run/lock                                                                                
tmpfs           492M     0  492M   0% /sys/fs/cgroup                                                                           
/dev/loop0       18M   18M     0 100% /snap/amazon-ssm-agent/1480                                                              
/dev/loop1       90M   90M     0 100% /snap/core/8268                                                                          
tmpfs            99M     0   99M   0% /run/user/1000     
```

**Note**: The root volume is now 20Gb

# Result!

The demo EC2 instance is now operational again - this time with a resized and smaller 20Gb gp2 root ebs volume.

:metal::metal::metal: