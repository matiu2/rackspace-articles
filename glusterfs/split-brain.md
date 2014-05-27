## Split brain

    root@matt:~# gluster volume heal www full
    Commit failed on 192.168.0.1. Please check the log file for more details.
    root@matt:~# tail /var/log/glusterfs/*log | grep E
    [2014-05-26 10:56:36.469159] I [input.c:36:cli_batch] 0-: Exiting with: 0
    [2014-05-26 10:56:43.481376] I [input.c:36:cli_batch] 0-: Exiting with: -1
    [2014-05-26 10:56:43.480537] E [glusterd-rpc-ops.c:890:__glusterd_commit_op_cbk] 0-management: Received commit RJT from uuid: 8555eac6-de14-44f6-babe-f955ebc16646
    [2014-05-26 10:56:43.928579] E [afr-self-heal-common.c:197:afr_sh_print_split_brain_log] 0-www-replicate-1: Unable to self-heal contents of '/etc/etc/cron.daily/apt' (possible split-brain). Please delete the file from all but the preferred subvolume.- Pending matrix:  [ [ 0 1 ] [ 1 0 ] ]


    root@matt:~# gluster volume heal www info split-brain
    Gathering Heal info on volume www has been successful

    Brick 192.168.0.1:/srv/.bricks/www
    Number of entries: 0

    Brick 192.168.0.3:/srv/.bricks/www
    Number of entries: 0

    Brick 192.168.0.2:/srv/.bricks/www
    Number of entries: 4
    at                    path on brick
    -----------------------------------
    2014-05-26 10:58:17 /etc/etc/cron.daily/apt
    2014-05-26 10:56:43 /etc/etc/cron.daily/apt
    2014-05-26 10:52:59 /etc/etc/cron.daily/apt
    2014-05-26 10:42:59 /etc/etc/cron.daily/apt

    Brick 192.168.0.4:/srv/.bricks/www
    Number of entries: 2
    at                    path on brick
    -----------------------------------
    2014-05-26 10:52:59 /etc/etc/cron.daily/apt
    2014-05-26 10:42:59 /etc/etc/cron.daily/apt

### on Web02 (the version I don't want)

BRICK=/srv/.bricks/www
SBFILE=/etc/etc/cron.daily/apt
GFID=$(getfattr -n trusted.gfid --absolute-names -e hex ${BRICK}${SBFILE} | grep 0x | cut -d'x' -f2)
rm -f ${BRICK}${SBFILE}
rm -f ${BRICK}/.glusterfs/${GFID:0:2}/${GFID:2:2}/${GFID:0:8}-${GFID:8:4}-${GFID:12:4}-${GFID:16:4}-${GFID:20:12}

restart 

## On Web01

root@matt:~# gluster volume heal www info split-brain
Gathering Heal info on volume www has been successful

Brick 192.168.0.1:/srv/.bricks/www
Number of entries: 0

Brick 192.168.0.3:/srv/.bricks/www
Number of entries: 0

Brick 192.168.0.2:/srv/.bricks/www
Number of entries: 0

Brick 192.168.0.4:/srv/.bricks/www
Number of entries: 3
at                    path on brick
-----------------------------------
2014-05-26 11:02:59 /etc/etc/cron.daily/apt
2014-05-26 10:52:59 /etc/etc/cron.daily/apt
2014-05-26 10:42:59 /etc/etc/cron.daily/apt

