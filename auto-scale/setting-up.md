# Autoscale - Setting Up

## Setting up

Lets start to build out the archetecture.

### Network

Using the [nova API](http://docs.rackspace.com/servers/api/v2/cs-gettingstarted/content/nova_create_server.html) create a 1 GB Ubuntu Trusty server.

First we'll create the network:

    nova network-create autoscale 192.168.0.0/24

### DB Servere

We'll build the DB server; it will have the network address 192.168.0.1:

    nova boot --image bb02b1a3-bc77-4d17-ab5b-421d89850fca --flavor performance1-1 --nic net-id=940e0093-ba6b-44f9-b3af-e24556293e9d db

Replace '940e0093-ba6b-44f9-b3af-e24556293e9d' with the UUID of the network as returned by the network-create command above.

We'll run these comamands on the DB server to set it up:

    apt-get update
    apt-get install mysql-server mysql-client
    ufw allow in on eth2
    mysql -p -e "create database wordpress"
    mysql -p -e "grant all on wordpress.* to wordpress@'192.168.0.%' identified by 'wordpress'"
    # Make mysql available on our network
    vim /etc/mysql/my.cnf 
    sed -i s/127.0.0.1/0.0.0.0/g /etc/mysql/my.cnf

We won't cover the DB slave server, not setting up DB backups in this article. It is left as an exercise for the reader.

Also, this is a very simple example; on a real life server we would certainly lock this down a bit more.

### Web 01

We'll create an initial server and use the [Gluster FS](TODO: link these) articles to set it up serving wordpress.

To create the 1st server with nova:

    nova boot --image bb02b1a3-bc77-4d17-ab5b-421d89850fca --flavor performance1-1 --nic net-id=940e0093-ba6b-44f9-b3af-e24556293e9d web01

It will have the network IP address 192.168.0.2:

These are the comamnds Jo ran on Web01 in our case study

    apt-get update
    apt-get install glusterfs-client glusterfs-server wordpress apache2 -y
    mkdir -p /.bricks
    gluster volume create www transport tcp 192.168.0.2:/.bricks/www force 
    gluster volume start www
    rm -rf /var/www
    mkdir /var/www
    echo localhost:/www /var/www glusterfs defaults,_netdev 0 0 >> /etc/fstab
    mount /var/www
    mkdir /var/www/html
    rsync -a /usr/share/wordpress/ /var/www/html/
    ufw allow http
    ufw allow in on eth2
    # Configure wordpress
    rm /var/www/html/wp-config.php
    cp /var/www/html/wp-config-sample.php  /var/www/html/wp-config.php
    sed -i s/database_name_here/wordpress/g  /var/www/html/wp-config.php 
    sed -i s/username_here/wordpress/g  /var/www/html/wp-config.php 
    sed -i s/password_here/wordpress/g  /var/www/html/wp-config.php 
    sed -i s/localhost/192.168.0.1/g  /var/www/html/wp-config.php 

## Web 02

Now we'll build Web02, that will be similar to Web01, but it'll become the image that's used for all autoscale builds, so it'll need a startup script that makes it join the glusterfs network.

    nova boot --image bb02b1a3-bc77-4d17-ab5b-421d89850fca --flavor performance1-1 --nic net-id=940e0093-ba6b-44f9-b3af-e24556293e9d web02

We'll copy some of the commands from Web01, but then we need to add a bootup script:

    apt-get update
    apt-get install glusterfs-client glusterfs-server wordpress apache2 -y
    mkdir -p /.bricks

