# Glusterfs - Coping with a downed server

## Prerequisites

You will have read our [previous article](TODO: MAKE LINK HERE) and have a 2-4 node gluster fs array, and know how to add and delete nodes.

For the purpose of this article, I'm running on a 4 node fully replicated gluster volume.

I've filled my glusterfs array with fake data for the testing.

## Overview

We'll show two forms of disaster recovery here:

 1. A single node wend down, and we're adding a new node to take its place.
 2. A single node went down, got rebuilt and kept the IP - this turns out to be more work to fix

## Node down, adding a replacement node

In this scenario, web03 will go down again, but we'll add a new node at 192.168.0.5 to replace it. This method is much easier.

Running from one of the good nodes, add the new sever into the mix:

    root@matt:~# gluster peer probe 192.168.0.5
    peer probe: success

Swap out the bad brick for the new one:

    root@matt:~# gluster volume replace-brick www 192.168.0.3:/srv/.bricks/www 192.168.0.5:/srv/.bricks/www  commit force
    volume replace-brick: success: replace-brick commit successful

Heal the system:

    root@matt:~# gluster volume heal www full
    Launching Heal operation on volume www has been successful
    Use heal info commands to check status

Watch it work:

    root@matt:~# gluster volume heal www info
    Gathering Heal info on volume www has been successful
    ...
    Brick 192.168.0.4:/srv/.bricks/www
    Number of entries: 23
    /wordpress/wp-admin/upload.php

If you were running a distributed system, you'll want to run:

    root@matt:~# gluster volume rebalance www fix-layout start
    volume rebalance: www: success: Starting rebalance on volume www has been successful.
    ID: 0a9719c1-cf04-4161-b3b0-cc6fd8dd9108
    root@matt:~# gluster volume rebalance www status
                                        Node Rebalanced-files          size       scanned      failures       skipped         status run time in secs
                                   ---------      -----------   -----------   -----------   -----------   -----------   ------------   --------------
                                   localhost                0        0Bytes             0             0             0      completed             1.00
                                   localhost                0        0Bytes             0             0             0      completed             1.00
                                 192.168.0.2                0        0Bytes             0             0             0      completed             1.00
                                 192.168.0.4                0        0Bytes             0             0             0      completed             1.00
                                 192.168.0.4                0        0Bytes             0             0             0      completed             1.00
                                 192.168.0.5                0        0Bytes             0             0             0      completed             1.00
    volume rebalance: www: success:

## Node down, keeping the IP address

In this scenario, server 3, with the IP address 192.168.0.3 has crashed and is completely unrecoverable.

We'll build a new machine, with the *same IP address,* trick glusterfs into thinking it's the old one, and let it self heal.

and simply rebalance it into the glusterfs.  You can refer to the previous articles for information on building and configuring the replacement node.

### Disguising the new web03 as the old one

Now that the new machine is built, glusterfs is installed on it, and the disk space has been prepared for the brick, we need to give it the peer UUID of the old server. To get the UUID, run this on one of the good nodes:

    root@web01:~# grep 192.168.0.3 /var/lib/glusterd/peers/*
    /var/lib/glusterd/peers/ba502dc2-447f-466a-a732-df989e71b551:hostname1=192.168.0.3

Copy the file name (which is the original Web03 UUID); in my case it is: ba502dc2-447f-466a-a732-df989e71b551 .

We'll assign the failed server's UUID to the new server. First we need to stop the gluster daemon:

    root@web03:~# service glusterfs-server stop
    glusterfs-server stop/waiting

Next replace the generated node UUID, with the new one in the glusterd config file:

    root@web03:~# UUID=ba502dc2-447f-466a-a732-df989e71b551
    root@web03:~# sed  -i "s/\(UUID\)=\(.*\)/\1=$UUID/g" /var/lib/glusterd/glusterd.info 
    root@web03:~# cat /var/lib/glusterd/glusterd.info 
    UUID=ba502dc2-447f-466a-a732-df989e71b551
    operating-version=2

**NOTE:** The ba502dc2... is from my system, you *need* to replace this with the old UUID from your old (downed) server (as remembered by Web01).

Start the server again:

    root@web03:~# service glusterfs-server start
    glusterfs-server start/running, process 10732

### Getting the peers re-configured

On the new node, check that the other peers are visible:

    root@web03:~# gluster peer status
    peer status: No peers present

If they're not, you have to add them explicitly:

    root@web03:~# gluster peer probe 192.168.0.1 
    peer probe: success
    root@web03:~# gluster peer probe 192.168.0.2
    peer probe: success
    root@web03:~# gluster peer probe 192.168.0.4
    peer probe: success

Now if you run `gluster peer status` now on Web03 it will say: `State: Accepted peer request (Connected)` - just give the daemon one more restart and the peers will be happy again:

    root@web03:~# service glusterfs-server restart
    glusterfs-server stop/waiting
    glusterfs-server start/running, process 9123
    root@web03:~# gluster peer status
    Number of Peers: 3

    Hostname: 192.168.0.2
    Uuid: 177cd473-9421-4651-8d6d-18be3a7e1990
    State: Peer in Cluster (Connected)

    Hostname: 192.168.0.1
    Uuid: 8555eac6-de14-44f6-babe-f955ebc16646
    State: Peer in Cluster (Connected)

    Hostname: 192.168.0.4
    Uuid: 1681b266-dc31-42e1-ab82-4e220906eda1
    State: Peer in Cluster (Connected)

### Syncing the volumes

Check the volume status:

    root@web03:~# gluster volume status
    No volumes present

We'll need to get those from a peer:

    root@web03:~# gluster volume sync 192.168.0.2 all
    Sync volume may make data inaccessible while the sync is in progress. Do you want to continue? (y/n) y
    volume sync: success

Finally we have to get the file system for the brick into order. In my example the brick is stored in /srv/.bricks/www:

    root@web03:~# mkdir /srv/.bricks/www

Go to one of the good nodes, install 'attr' and get the correct volume id, like so:

    root@web02:~# apt-get install attr -y
    ...
    root@web02:~# getfattr  -n trusted.glusterfs.volume-id /srv/.bricks/www
    getfattr: Removing leading '/' from absolute path names
    # file: srv/.bricks/www
    trusted.glusterfs.volume-id=0s42V5HW+LSuyzqotW1jgAhA==

So in this example, we'll want to copy that string to our clipboard: 0s42V5HW+LSuyzqotW1jgAhA==

Now on the replacement node, apply that extended attribute:

    root@web03:~# apt-get install attr -y
    ...
    root@web03:~# setfattr -n trusted.glusterfs.volume-id -v '0s42V5HW+LSuyzqotW1jgAhA==' /srv/.bricks/www
    
Nearly there. One more restart, then a heal and all will be well with the world again:

    root@matt:~# service glusterfs-server restart
    glusterfs-server stop/waiting
    glusterfs-server start/running, process 13318
    root@matt:~# gluster volume heal www full
    Launching Heal operation on volume www has been successful
    Use heal info commands to check status

Now everything should be good with the world again:

    root@matt:~# gluster volume heal www info
    Gathering Heal info on volume www has been successful

    Brick 192.168.0.1:/srv/.bricks/www
    Number of entries: 0

    Brick 192.168.0.2:/srv/.bricks/www
    Number of entries: 0

    Brick 192.168.0.3:/srv/.bricks/www
    Number of entries: 0

    Brick 192.168.0.4:/srv/.bricks/www
    Number of entries: 0

--------------------------------------

# Conclusion
