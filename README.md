# Articles for submission to the Rackspace Knowledge base

## Glusterfs

 1. [How to set up a 2 node glusterfs array](glusterfs/how-to-setup-a-two-node-glusterfs-array.md)
 2. [How to add and remove a node](glusterfs/how-to-add-and-remove-a-node.md)
 3. [How to cope with a downed server](glusterfs/coping-with-a-downed-server.md)

### Bits and pieces

[Split Brain notes](glusterfs/split-brain.md)
[Some benchmarks](glusterfs/timing.md)

# Auto Scale

1. [Overview](auto-scale/advantages.md) - Advantages to scaling out rather than up. Choosing an archetecture, making sure not to overload the DB(s), using read-only DB slaves, how to share php and magento session info between nodes(glusterfs/memcached), how to share and handle files and file writes (glusterfs/lsyncd+varnish/nginx)
2. Setting up auto scaling - link to docs, creating the initial image, choosing the trigger levels, server limits. Implementing the API calls, documenting your script.
3. Testing autoscale - load testing, getting a notification when it's activated analysing LB and server logs to measure effectiveness.
