=================
Quick Start Guide
=================

For this tutorial, we will assume you are using Fedora 22 (or later)
virtual machine(s). If you would like a more detailed walk through with
instructions for installing using different methods (in local virtual
machines, EC2 and baremetal) and different distributions, then have a look
at the Install Guide.


Single Node Cluster
^^^^^^^^^^^^^^^^^^^
This is to demonstrate installation and setting up of GlusterFS in under five
minutes. You would not want to do this in any real-world scenario.

Install glusterfs client and server packages:

::

    # yum install glusterfs glusterfs-server glusterfs-fuse

Start glusterd service:

::

    # service glusterd start

Create 4 loopback devices to be consumed as bricks. This exercise is to
simulate 4 hard disks in 4 different nodes. Format the loopback devices with
XFS filesystem and mount it.

::

    # truncate -s 1GB /srv/disk{1..4}
    # for i in `seq 1 4`;do mkfs.xfs -i size=512 /srv/disk$i ;done
    # mkdir -p /export/brick{1..4}
    # for i in `seq 1 4`;do echo "/srv/disk$i /export/brick$i   xfs   loop,inode64,noatime,nodiratime 0 0" >> /etc/fstab ;done
    # mount -a

Create a 2x2 Distributed-Replicated volume and start it:

::

    # gluster volume create test replica 2 transport tcp `hostname`:/export/brick{1..4}/data force
    # gluster volume start test

Mount the volume for consumption:

::

    # mkdir /mnt/test
    # mount -t glusterfs `hostname`:test /mnt/test

For illustration, create 10 empty files and see how it gets distributed and
replicated among the 4 bricks that make up the volume.

::

    # touch /mnt/test/file{1..10}
    # ls /mnt/test
    # tree /export/

Multi Node Cluster
^^^^^^^^^^^^^^^^^^

Step 1 – Have at least two nodes
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

-  Fedora 22 (or later) on two nodes named "server1" and "server2"
-  A working network connection
-  At least two virtual disks, one for the OS installation, and one to
   be used to serve GlusterFS storage (sdb). This will emulate a real
   world deployment, where you would want to separate GlusterFS storage
   from the OS install.

Step 2 - Format and mount the bricks
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

(on both nodes): Note: These examples are going to assume the brick is
going to reside on /dev/sdb1.

::

        # mkfs.xfs -i size=512 /dev/sdb1
        # mkdir -p /data/brick1
        # echo '/dev/sdb1 /data/brick1 xfs defaults 1 2' >> /etc/fstab
        # mount -a && mount

You should now see sdb1 mounted at /data/brick1

Step 3 - Installing GlusterFS
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

(on both servers) Install the software

::

        # yum install glusterfs-server

Start the GlusterFS management daemon:

::

        # service glusterd start
        # service glusterd status

Step 4 - Configure the trusted pool
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

From "server1"

::

        # gluster peer probe server2

Note: When using hostnames, the first server needs to be probed from
**one** other server to set its hostname.

From "server2"

::

        # gluster peer probe server1

Note: Once this pool has been established, only trusted members may
probe new servers into the pool. A new server cannot probe the pool, it
must be probed from the pool.

Step 5 - Set up a GlusterFS volume
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

From any single server:

::

        # gluster volume create gv0 replica 2 server1:/data/brick1/gv0 server2:/data/brick1/gv0
        # gluster volume start gv0

Confirm that the volume shows "Started":

::

        # gluster volume info

Note: If the volume is not started, clues as to what went wrong will be
in log files under /var/log/glusterfs on one or both of the servers -
usually in etc-glusterfs-glusterd.vol.log

Step 6 - Testing the GlusterFS volume
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For this step, we will use one of the servers to mount the volume.
Typically, you would do this from an external machine, known as a
"client". Since using this method would require additional packages to
be installed on the client machine, we will use one of the servers as a
simple place to test first, as if it were that "client".

::

        # mount -t glusterfs server1:/gv0 /mnt
        # for i in `seq -w 1 100`; do cp -rp /var/log/messages /mnt/copy-test-$i; done

First, check the mount point:

::

        # ls -lA /mnt | wc -l

You should see 100 files returned. Next, check the GlusterFS mount
points on each server:

::

        # ls -lA /data/brick1/gv0

You should see 100 files on each server using the method we listed here.
Without replication, in a distribute only volume (not detailed here),
you should see about 50 files on each one.
