.. _cephadm-adoption:

Converting an existing cluster to cephadm
=========================================

Cephadm allows you to convert an existing Ceph cluster that
has been deployed with ceph-deploy, ceph-ansible, DeepSea, or similar tools.

Limitations
-----------

* Cephadm only works with BlueStore OSDs.  If there are FileStore OSDs
  in your cluster you cannot manage them.

Preparation
-----------

#. Get the ``cephadm`` command line tool on each host in the existing
   cluster.  See :ref:`get-cephadm`.

#. Prepare each host for use by ``cephadm``:

   .. prompt:: bash #

      cephadm prepare-host

#. Determine which Ceph version you will use.  You can use any Octopus (15.2.z)
   release or later.  For example, ``docker.io/ceph/ceph:v15.2.0``.  The default
   will be the latest stable release, but if you are upgrading from an earlier
   release at the same time be sure to refer to the upgrade notes for any
   special steps to take while upgrading.

   The image is passed to cephadm with:

   .. prompt:: bash #

      cephadm --image $IMAGE <rest of command goes here>

#. Cephadm can provide a list of all Ceph daemons on the current host:

   .. prompt:: bash #

      cephadm ls

   Before starting, you should see that all existing daemons have a
   style of ``legacy`` in the resulting output.  As the adoption
   process progresses, adopted daemons will appear as style
   ``cephadm:v1``.


Adoption process
----------------

#. Make sure that the ceph configuration has been migrated to use the cluster
   config database.  If the ``/etc/ceph/ceph.conf`` is identical on each host,
   then the following command can be run on one single host and will affect all
   hosts:

   .. prompt:: bash #

      ceph config assimilate-conf -i /etc/ceph/ceph.conf

   If there are configuration variations between hosts, you will need to repeat
   this command on each host. During this adoption process, view the cluster's
   configuration to confirm that it is complete by running the following
   command:

   .. prompt:: bash #

      ceph config dump

#. Adopt each monitor:

   .. prompt:: bash #

      cephadm adopt --style legacy --name mon.<hostname>

   Each legacy monitor should stop, quickly restart as a cephadm
   container, and rejoin the quorum.

#. Adopt each manager:

   .. prompt:: bash #

      cephadm adopt --style legacy --name mgr.<hostname>

#. Enable cephadm:

   .. prompt:: bash #

      ceph mgr module enable cephadm
      ceph orch set backend cephadm

#. Generate an SSH key:

   .. prompt:: bash #

      ceph cephadm generate-key
      ceph cephadm get-pub-key > ~/ceph.pub

#. Install the cluster SSH key on each host in the cluster:

   .. prompt:: bash #

      ssh-copy-id -f -i ~/ceph.pub root@<host>

   .. note::
     It is also possible to import an existing ssh key. See
     :ref:`ssh errors <cephadm-ssh-errors>` in the troubleshooting
     document for instructions that describe how to import existing
     ssh keys.

#. Tell cephadm which hosts to manage:

   .. prompt:: bash #

      ceph orch host add <hostname> [ip-address]

   This will perform a ``cephadm check-host`` on each host before adding it;
   this check ensures that the host is functioning properly. The IP address
   argument is required only if DNS does not allow you to connect to each host
   by its short name.

#. Verify that the adopted monitor and manager daemons are visible:

   .. prompt:: bash #

      ceph orch ps

#. Adopt all OSDs in the cluster:

   .. prompt:: bash #

      cephadm adopt --style legacy --name <name>

   For example:

   .. prompt:: bash #

      cephadm adopt --style legacy --name osd.1
      cephadm adopt --style legacy --name osd.2

#. Redeploy MDS daemons by telling cephadm how many daemons to run for
   each file system. List file systems by name with the command ``ceph fs
   ls``. Run the following command on the master nodes to redeploy the MDS
   daemons:

   .. prompt:: bash #

      ceph orch apply mds <fs-name> [--placement=<placement>]

   For example, in a cluster with a single file system called `foo`:

   .. prompt:: bash #

      ceph fs ls

   .. code-block:: bash

      name: foo, metadata pool: foo_metadata, data pools: [foo_data ]

   .. prompt:: bash #

      ceph orch apply mds foo 2

   Confirm that the new MDS daemons have started:

   .. prompt:: bash #

      ceph orch ps --daemon-type mds

   Finally, stop and remove the legacy MDS daemons:

   .. prompt:: bash #

      systemctl stop ceph-mds.target
      rm -rf /var/lib/ceph/mds/ceph-*

#. Redeploy RGW daemons. Cephadm manages RGW daemons by zone. For each
   zone, deploy new RGW daemons with cephadm:

   .. prompt:: bash #

      ceph orch apply rgw <svc_id> [--rgw-realm=<realm>] [--rgw-zone=<zone>] [--port=<port>] [--ssl] [--placement=<placement>]

   where *<placement>* can be a simple daemon count, or a list of
   specific hosts (see :ref:`orchestrator-cli-placement-spec`), and the
   zone and realm arguments are needed only for a multisite setup.

   After the daemons have started and you have confirmed that they are
   functioning, stop and remove the old, legacy daemons:

   .. prompt:: bash #

      systemctl stop ceph-rgw.target
      rm -rf /var/lib/ceph/radosgw/ceph-*

#. Check the output of the command ``ceph health detail`` for cephadm warnings
   about stray cluster daemons or hosts that are not yet managed by cephadm.
