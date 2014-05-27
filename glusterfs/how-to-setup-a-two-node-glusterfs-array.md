# How to set up a two node glusterfs array

## Reasoning

Having two web server machines behind a load balancer means they have to synchronize the files that they serve and write to. On modern Linux distros Glusterfs is the easiest way to do this. In this article we are using Ubuntu Trusty as the Linux distro.

## Disclaimer

This article uses an example set up using brand new machines. You'll need to alter some of the commands if you already have boxes set up, or to suit your usage. Parts of this article involve formatting disks and removing files, which could erase data; it is your responsibility to make sure that the disks you format and files you erase, don't contain important data before running the commands.

## Prerequisites

You'll need two Rackspace Cloud Machines running Ubuntu Trusty. In this article we'll use the new 'performance' flavor machines; the data disk's main partition /dev/xvde1 will be set apart for glusterfs.

If you're using [nova](http://docs.rackspace.com/servers/api/v2/cs-gettingstarted/content/section_gs_install_nova.html) you can build two 4 GB Performance severs with Ubuntu Trusty and PVHVM:

    nova boot --image bb02b1a3-bc77-4d17-ab5b-421d89850fca --flavor performance1-4 web1
    nova boot --image bb02b1a3-bc77-4d17-ab5b-421d89850fca --flavor performance1-4 web2

## Installation

glusterfs is really easy to install; just run this on both machines:

    apt-get update
    apt-get install -y glusterfs-server glusterfs-client

## Preparing the bricks

glusterfs needs a file system that supports extended attributes to store its data; It creates directories there and calls those directories *bricks.*jj

On our Rackspace Cloud 4 GB performance server, we have a whole extra drive that's already partitioned for us.

    root@web01:~# ls /dev/xvde*
    /dev/xvde  /dev/xvde1

We'll format this partition with ext4; run these commands on both servers:

    mkfs.ext4 /dev/xvde1

We don't want anything other than glusterfs using this partition, so we'll mount it in a hidden directory. I like to put them in '/srv/.bricks'. If you have multiple xfs volumes per server, you can put them in '/srv/.bricks1' etc.

    mkdir /srv/.bricks
    echo /dev/xvde1 /srv/.bricks ext4 defaults 0 1 >> /etc/fstab
    mount /srv/.bricks

I call it 'bricks' because each directory in there will be a glusterfs brick. A glusterfs volume is built of bricks (usually bricks on different hosts).

## Setting up a Rackspace network

We'll run glusterfs on it's own [Rackspace cloud network](http://www.rackspace.com/knowledge_center/article/getting-started-with-cloud-networks). This will allow us to manage the network and firewall settings much more easily.

I've set up a network using the nova client on my laptop:

    nova network-create glusterfs 192.168.0.0/24
    nova network-associate-host UUID 192.168.0.1
    nova network-associate-host UUID web02

The first command will return you the UUID of the network, which you can copy and paste into the subsequent commands, replacing 'UUID' with your network's actual UUID string. Also, replace '192.168.0.1' and 'web02' with the IP addresses or host names of your VMs, for example:

    nova network-associate-host 4dad2eb0-5ed7-4147-8196-bba7dc2bb45f 23.253.156.109

You could also do all these via [the web UI](http://www.rackspace.com/knowledge_center/article/getting-started-with-cloud-networks) instead of via the nova command line.

### Firewall

Next, we'll open up the firewall to allow all traffic on this network. Run this command on both servers:

    ufw allow in on eth2

In our example the Rackspace Cloud network is on the device 'eth2'; you can use the command `ifconfig` to see all network devices and networks associated with them.

If you added web01 to the network first, it'll have the IP address 192.168.0.1 and web02 will have 192.168.0.2.

## Glusterfs handshake

Let's introduce the two gluster servers to each other. Here I'm running the command on web01, and telling it to hook up with web02:

    root@web01:~# gluster peer probe web02
    peer probe: success

Now you can see that they are linked by running `gluster peer status` on web02:

    root@web02:~# gluster peer status
    Number of Peers: 1

    Hostname: 192.168.0.1
    Port: 24007
    Uuid: d080d5cc-4181-4d3f-91bc-ef42bb4e8ec9
    State: Peer in Cluster (Connected)

## Creating the glusterfs volumes

Finally we can create the volumes. Run this command only on one of the machines:

    root@web01:~# gluster volume create www replica 2 transport tcp 192.168.0.1:/srv/.bricks/www 192.168.0.2:/srv/.bricks/www
    volume create: www: success: please start the volume to access data

Lets break the command down:

 * gluster - this is the gluster command line tool
 * volume create - We are creating a gluster volume
 * www - This is the volume name. You can call it whatever you like. It will be used later in your /etc/fstab when mounting the volume.
 * replica 2 - Have every file on this volume be replicated between 2 bricks; in our case that means at least two servers (as we only have one brick per server).
 * transport tcp - Use TCP/IP to synchronize the volumes; it's our only practical option.
 * 192.168.0.1:/srv/.bricks/www - this is the first brick with which the volume will be built (on web01)
 * 192.168.0.2:/srv/.bricks/www - this is the second brick with which the volume will be built, (on web02)

For more information on these options you can run `man gluster`.

## Start and mount the volume

The volume exists, but is not being actively synchronized nor served. Let's start it up. Run this command on either machine:

    root@web01:~# gluster volume start www
    volume start: www: success

We'll mount it in /srv/www initially. Run these commands on both servers:

    mkdir /srv/www
    echo localhost:/www /srv/www glusterfs defaults,_netdev 0 0 >> /etc/fstab
    mount /srv/www

Here, we create the mount point, then configure it in /etc/fstab, then actually mount the glusterfs volume.

In /etc/fstab we add one special option `_netdev`; this tells Ubuntu that the filesystem resides on a device that requires network access, and not to try and mount it until the network has been enabled.

## Testing it out

At this point a file write or read to /srv/www/* should be the same on both systems.

To test it out, create a file on web01, view it on web02, then delete it on web02 and see it gone on Web01:

Web01:

    echo hello > /srv/www/test.txt

Web02:

    cat /srv/www/test.txt  # should print 'hello'
    rm /srv/www/test.txt   # Delete the file from Web02

Web01:

    ls /srv/www/ # Should return nothing

## Move your web content to glusterfs

In this example, we want to have /var/www be on glusterfs.  I'm assuming that web01 has the correct /var/www.

If you are running this on an already live machine, you'll want to shut down Apache on both machines. You could set up a custom 'down for maintenance page' and health monitoring on the [Load balancer](http://www.rackspace.com/knowledge_center/article/rackspace-cloud-essentials-configuring-a-load-balancer) first if you like, but that's beyond the scope of this article.

On **Web01**, lets move /var/www/ to /srv/www/:

    mv /var/www/* /srv/www/

On **Web02**, if you're sure you don't need it, you can free up space by deleting /var/www:

    rm -rf /var/www
    mkdir /var/www

Now, we'll just create a bindmount so that /srv/www is accesible via /var/www. Run this on **both** machines:

    echo /srv/www /var/www none defaults,bind 0 0 >> /etc/fstab
    mount /var/www

You should be able to see your web content with `ls /var/www` on both machines.

### Summary

Your /etc/fstab should look like this on both machines:

    # <file system> <mount point>   <type>  <options>       <dump>  <pass>
    /dev/xvda1	/               ext4    errors=remount-ro,noatime,barrier=0 0       1
    /dev/xvde1 /srv/.bricks ext4 defaults 0 1
    localhost:/www /srv/www glusterfs defaults,_netdev 0 0
    /srv/www /var/www none defaults,bind 0 0

If you run `findmnt | grep srv` it should look something like this:

    root@web01:~# findmnt | tail -n3
    ├─/srv/.bricks               /dev/xvde1     ext4            rw,relatime,attr2,inode64,noquota
    ├─/srv/www                   localhost:/www fuse.glusterfs rw,relatime,user_id=0,group_id=0,default_permissions,allow_other,max_read=131072
    └─/var/www                   localhost:/www fuse.glusterfs rw,relatime,user_id=0,group_id=0,default_permissions,allow_other,max_read=131072

This shows that /srv/.bricks is mounted on /dev/xvde1 and that localhost:/www (the glusterfs volume) is mounted in two places, /srv/www and /var/www (thanks to the magic of bind mounting).

glusterfs should show everything as healthy:

    root@web01:~# gluster peer list
    unrecognized word: list (position 1)
    root@web01:~# gluster peer status
    Number of Peers: 1

    Hostname: web02
    Port: 24007
    Uuid: 56e02356-d2c3-4787-ae25-6b46e867751a
    State: Peer in Cluster (Connected)

    root@web01:~# gluster volume list
    www

    root@web01:~# gluster volume info www
    Volume Name: www
    Type: Replicate
    Volume ID: bf244b65-4201-4d2f-b8c0-2b11ee836d65
    Status: Started
    Number of Bricks: 1 x 2 = 2
    Transport-type: tcp
    Bricks:
    Brick1: 192.168.0.1:/srv/.bricks/www
    Brick2: 192.168.0.2:/srv/.bricks/www

## Conclusion

Well Done! You've installed glusterfs and configured your servers to share your web content. Both servers hold a copy of the files and share changes pretty much instantaneously.

In the upcoming articles, we'll investigate adding a server to the cluster, removing a server to the cluster and disaster recovery.

 - matiu
