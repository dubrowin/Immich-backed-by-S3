# Immich-backed-by-S3
How I built an Immich server backed by S3
![Immich drawio](https://github.com/user-attachments/assets/5da32b14-4d30-4730-abe9-348c3bf002c7)

## What is Immich

[Immich](https://immich.app/) is an open source project to manage photo albums. Recently, my son's account for storage was filling up and the provider was offering a higher tier to pay for storage. Rather than pay for their storage, we downloaded all his photos and then uploaded them to an S3 bucket. Now, I wanted to give him better access to the pictures.

## Why Back it by S3

S3 is the cheapest and safest storage available on AWS. If I wanted to run the Immich server on EC2 and pay for the storage, I would need around 200GB minimum and that would cost me just in GP3 storage without snapsots $16 per month (in the Ohio Region). The same storage in S3 Standard Tier costs $4.60. S3 is also multi-AZ with many 9's of redundancy. And I can automate the simple addition of files (S3 objects) via other scripts.

## Steps for my build

I started with a t4g.micro instance for the [install](https://immich.app/docs/overview/quick-start) process. For installation, this worked fine, but the docker containers wouldn't run. To get the system up and running, I used a t4g.medium. Once everything was setup, I downgraded to a t4g.small. The small instance works, but I see there are times it pauses, this could be from network throttling or just the time it takes to communicate with S3 for the photo storage.

***Note:*** Throughout these examples, I talk about bucket-1 and bucket-2. These are also included in the mount-s3 file I use for the automatic mounting during boot. These are fake bucket names and you should replace these with your bucket names.

### MountPoint S3

AWS released [mountpoint-s3](https://github.com/awslabs/mountpoint-s3). I'm using this to mount the S3 buckets I need.

#### Fuse
At first, I was getting an error from mkdir during ```docker compose```. With some googling, I found on [Stack Overflow](https://stackoverflow.com/questions/50817985/docker-tries-to-mkdir-the-folder-that-i-mount) that I needed to allow other users access to the directories.

First I needed to uncomment in ```/etc/fuse.conf```
user_allow_other

And then, when using the ```mount-s3``` command (the command created from mountpoint-s3), I needed to include ```--allow-root``` since Docker is running as root.

**Experimental**
*I'm still testing and verifying this works as expected*

I even created a a [startup script](https://github.com/dubrowin/Immich-backed-by-S3/blob/main/mount-s3), which is still a work in progress.
- you will need to update the bucket-1 and/or bucket-2
- if you are not using Amazon Linux, you may need to update the USER that the process runs as
- you may desire a different directory for your mount points
- In order for this to start before docker (needed if Immich on Docker will be the R/W store), I needed to update ```/lib/systemd/system/docker.service```
  - to add ```mount-s3``` to the end of the ```After``` and ```Requires``` lines.
- I placed the ```mount-s3``` file in ```/etc/init.d/``` made sure it's executable
  - ```chmod +x /etc/init.d/mount-s3```
- And enable the script
  - ```sudo systemctl enable mount-s3```

***TODO*** mount-s3 is still starting after the docker containers. Therefore, on reboot, I need to stop the docker containers, run the mount-s3, and then restart the docker containers. Need to dive more into the systemd startup order.

### Deploy Immich

To configure Immich to use the S3 mountpoints, I needed to 
- adjust the ```.env``` file
```
UPLOAD_LOCATION=/home/ec2-user/mnt/bucket-1
```
This tells Immich to store all uploads into that directory which is mounted by mount-s3 to bucket-1

- I also had another bucket, I wanted to mount as read only, for that, I needed to adjust the ```docker-compose.yml``` file:
```
     - /home/ec2-user/mnt/bucket-2:/mnt/bucket-2:ro
```
In the volumes section, this line tells Docker to mount the bucket-2 mount point for mount-s3 as /mnt/bucket-2 and to make it read-only

In the Immich External Library Setup, I added /mnt/bucket-2 as a scan path to find new photos.

#### Library Scanning

In order to support the library scanning more efficiently, I upgraded the instance from ```t4g.small``` to ```c6g.xlarge``` for a few hours. While the instance costs more, once the scanning is complete, I'll shutdown and resize back dwon tot he ```t4g.small``` to support the app and website usages.

## Tailscale

I use [Tailscale](https://tailscale.com/) for my personal networking needs. This means there are no Security Groups needed for my devices to gain access to the Immich Server. I even renamed the tailscale network name of the device to Immich, so connecting to the server is by simple name.

## Next Steps
- I'm considering running the EC2 instance as spot so I can pay less
- I also need to monitor the S3 request costs to see if this style deployment makes sense
