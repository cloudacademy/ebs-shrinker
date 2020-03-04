# ebs-shrinker

```
REGION=us-west-2
AZ=us-west-2c
IMAGE_ID=ami-0d1cd67c26f5fca19 #ubuntu 18.04
USER=ubuntu
VOLUME_SIZE_LARGE=50 #starting size for ebs root volume
VOLUME_SIZE_SMALL=20 #final size after resizing for ebs root volume
```

You need to update the following environment variables which will be used later to establish a demo EC2 instance with an initial root volume of 50Gb 

```
SSH_KEY_NAME=<ec2 ssh keyname here - example: DemoSSHKey>
SSH_KEY_PATH=<ec2 ssh key path here - example: ~/.ssh/DemoSSHKey.pem>
SUBNET_ID=<subnet id here - where demo ec2 instance is deployed - example: subnet-aa598002>
SECURITY_GROUP_ID=<security group id here - needs to allow inbound ssh connections to the demo ec2 instance - example: >
```

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
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=EBSShrink}]" "ResourceType=volume,Tags=[{Key=Name,Value=EBSLargeVol}]"`
```

```
INSTANCE_ID=`echo $CREATE_INSTANCE_RESULT | jq -r .Instances[0].InstanceId`
```

```
aws ec2 describe-volumes \
    --region $REGION \
    --filters Name=attachment.instance-id,Values=$INSTANCE_ID
```

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

```
aws ec2 stop-instances  \
    --region $REGION \
    --instance-ids $INSTANCE_ID

aws ec2 wait instance-stopped \
    --region $REGION \
    --instance-ids $INSTANCE_ID
```

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

PUBLIC_IP=`aws ec2 describe-instances \
    --region $REGION \
    --filters \
    "Name=instance-state-name,Values=running" \
    "Name=instance-id,Values=$INSTANCE_ID" \
    --query 'Reservations[*].Instances[*].[PublicIpAddress]' \
    --output text`

echo PUBLIC_IP=$PUBLIC_IP
```

```
ssh -o "StrictHostKeyChecking no" -v -i $SSH_KEY_PATH $USER@$PUBLIC_IP 'bash -s' <<\EOF
lsblk
echo
df -h
EOF
```

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

```
aws ec2 stop-instances  \
    --region $REGION \
    --instance-ids $INSTANCE_ID

aws ec2 wait instance-stopped \
    --region $REGION \
    --instance-ids $INSTANCE_ID
```

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

```
ssh -o "StrictHostKeyChecking no" -v -i $SSH_KEY_PATH $USER@$PUBLIC_IP 'bash -s' <<\EOF
lsblk
echo
df -h
EOF
```