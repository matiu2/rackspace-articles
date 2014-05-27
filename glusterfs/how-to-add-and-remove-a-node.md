# Gluster FS -  How to add and remove a node

## Prerequisites

Following on from our [previous article](TODO: MAKE LINK HERE), you'll now have a two node glusterfs array.

In this article we'll add a new node, balance it into the array then remove it.

## Boot a new server

We'll use the [nova](http://docs.rackspace.com/servers/api/v2/cs-gettingstarted/content/section_gs_install_nova.html) boot command from the previous article to boot web3:

    nova boot --image bb02b1a3-bc77-4d17-ab5b-421d89850fca --flavor performance1-4 web3

You could also use the [Rackspace Cloud Control panel](http://www.rackspace.com/knowledge_center/article/introducing-the-new-cloud-control-panel) to create the new server.

## Add to the Rackspace custom network

In [previous article](TODO: MAKE LINK HERE) we added a Rackspace custom network, you can get it's UUID with this nova command:

    nova network-list

Once you have the UUID you can associate the new host to it. Replace UUID in the below command with the actual UUID (eg. 5492de89-1497-4aa0-96eb-bcdd55e1195c):

    nova network-associate-host UUID web03

Of course, 'web03' is the hostname of the node you wish to add.

You can also use the [Rackspace Cloud Control panel](http://www.rackspace.com/knowledge_center/article/introducing-the-new-cloud-control-panel) to associate a server with your existing network.

When done, the new server should have the IP address 192.168.0.3 on interface /dev/eth3. That's what glusterfs will use to communicate with the other server.

## Format the partition and install glusterfs

Using the instructions from the [previous article](TODO: MAKE LINK HERE), ssh in, install glusterfs, format the 'bricks' partition:

    apt-get update
    apt-get install -y glusterfs-server glusterfs-client
    apt-get install -y xfsprogs
    mkfs.xfs /dev/xvde1
    mkdir /srv/.bricks
    echo /dev/xvde1 /srv/.bricks xfs rw 0 1 >> /etc/fstab
    mount /srv/.bricks
    ufw allow in on eth2

## Incorporate the new brick into the gluster volume

ssh into Web01 or Web02 now (it doesn't matter which).

Tell glusterfs to trust the new server:

    root@web02 :~# gluster peer probe 192.168.0.3
    peer probe: success

Now just add the brick into the volume:

    root@web02 :~# gluster volume add-brick www replica 3 192.168.0.3:/srv/.bricks/www
    volume add-brick: success

Lets break the command down:

 * gluster - We're talking to glusterfs
 * volume - It's a command related to a volume
 * add-brick - We're adding a brick to the volume
 * www - We're working with the volume called 'www'
 * replica 3 - After adding this brick the volume will keep at least 3 copies of each file (one copy per brick, and in our case one copy per server (as we only have one brick on each server).
 * 192.168.0.3:/srv/.bricks/www - this is the IP address of the gluster server, followed by the absolute path to where the brick data should be stored.

### Different volume storage strategies

It's worth mentioning at this point, that glusterfs offers different types of volume storage strategies:

 * Distributed - One file on one brick, the next file on the next. This gives you more space as your volume is the sum of all the bricks.
 * Replicated - Every file is copied to every server. This is the only one I recommend using.
 * Striped - Files are cut into chunks, and one chunk is written to the 1st brick, then one to the second, and so on.

You can also combine the above, for example replicated-distributed:

    gluster volume create www replica 2 transport tcp 192.168.0.1:/srv/.bricks/www 192.168.0.2:/srv/.bricks/www 192.168.0.3:/srv/.bricks/www 192.168.0.4:/srv/.bricks/www

The replica number is the number of bricks to make up a replica set, ie. hold a full copy of the files. In the above example 192.168.0.1 + 192.168.0.2 hold a full copy of the files, as do 192.168.0.3 + 192.168.0.4. The brick order is significant; if you ordered it 1,3,2,4 then 1+3 and 2+4 would hold the full files, ie. if 1+2 went down, you'd loose half your files and have 2 copies of the other half.

Having a replicated-distributed volume gives you a little extra speed, and more space in exchange for data safety.

Striped-replicated volumes are only recommended when you have files that are bigger than a brick, or a lot of large files undergoing a lot of IO operations.

## Seeing the state of things

Lets have a look at some commands that we can use to get a better idea about what's going on in our cluster. You'll want to use these in the later articles.

### gluster peer status

If you run this command from any server, it'll show all the peer servers that it knows about:

    root@web01:~# gluster peer status
    Number of Peers: 2

    Hostname: 192.168.0.3
    Uuid: ba502dc2-447f-466a-a732-df989e71b551
    State: Peer in Cluster (Connected)

    Hostname: 192.168.0.2
    Uuid: 56e02356-d2c3-4787-ae25-6b46e867751a
    State: Peer in Cluster (Connected)

### Volume status

This is probably the best go-to trouble shooting command. It gives information about all the glusterfs volumes and tasks queued and in progress.

    root@web03:~# gluster volume status
    Status of volume: www
    Gluster process						Port	Online	Pid
    ------------------------------------------------------------------------------
    Brick 192.168.0.2:/srv/.bricks/www			49152	Y	13673
    Brick 192.168.0.1:/srv/.bricks/www			49152	Y	10249
    Brick 192.168.0.3:/srv/.bricks/www			49153	Y	13783
    NFS Server on localhost					2049	Y	13793
    Self-heal Daemon on localhost				N/A	Y	13800
    NFS Server on 192.168.0.2				2049	Y	13900
    Self-heal Daemon on 192.168.0.2				N/A	Y	13907
    NFS Server on 192.168.0.1				2049	Y	10286
    Self-heal Daemon on 192.168.0.1				N/A	Y	10293
     
    There are no active volume tasks

## Remove a brick

Lets remove brick 2 from the gluster:

    root@web01:~# gluster volume remove-brick www replica 2 192.168.0.2:/srv/.bricks/www
    Removing brick(s) can result in data loss. Do you want to Continue? (y/n) y
    volume remove-brick commit force: success

Here, we're telling glusterfs that the 'www' volume will now keep only 2 copies of each file. It's prompting us that we may loose data.

If you were on a distributed volume, you'd want to run the command like this:

    root@web01:~# gluster volume remove-brick www replica 2 192.168.0.2:/srv/.bricks/www start

Then keep an eye on it until it finishes:

    root@web01:~# watch gluster volume remove-brick www replica 2 192.168.0.2:/srv/.bricks/www status

That will give glusterfs time to re-distribute the files around the bricks.

## Re-add a brick

As re-adding is not as straight forward as it could be, I'll cover it here.

Lets try adding web02 back into the gluster:

    root@web01:~# gluster volume add-brick www replica 3 192.168.0.2:/srv/.bricks/www
    volume add-brick: failed: 

Oh no! It failed  .. Why is that ?  Lets have a look at the logs **on Web02**:

    root@web02:/srv/.bricks# tail /var/log/glusterfs/*log -f | grep E
    [2014-05-25 00:19:04.954410] I [input.c:36:cli_batch] 0-: Exiting with: 0
    [2014-05-25 00:19:12.958620] I [input.c:36:cli_batch] 0-: Exiting with: 0
    [2014-05-25 00:40:46.923747] E [glusterd-utils.c:5377:glusterd_is_path_in_use] 0-management: /srv/.bricks/www or a prefix of it is already part of a volume
    [2014-05-25 00:40:46.923789] E [glusterd-op-sm.c:3719:glusterd_op_ac_stage_op] 0-management: Stage failed on operation 'Volume Add brick', Status : -1

It's complaining because /srv/.bricks/www still contains the data from when web02 was a member of the volume.

We need to give it a clean place to store the data. The easiest way to clean up is to just remove it all:

    root@web02:~# rm -rf /srv/.bricks/www

**Be careful** to do this on the correct host (web02 that is currently out of the volume) ! If you do muck it up. The next article shows how to recover. Of course we could have just moved the 'www' dir out of the way just in case, or have added the brick using say 'www2' as the dir.

Now that we have a clean location to store the brick, adding the brick is nice and easy:

    root@web01:/srv# gluster volume add-brick www replica 3 192.168.0.2:/srv/.bricks/www
    volume add-brick: success

## Conclusion

In our next article we'll look at coping with a downed server.
