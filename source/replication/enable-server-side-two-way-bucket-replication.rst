.. _minio-bucket-replication-serverside-twoway:

=============================================
Enable Two-Way Server-Side Bucket Replication
=============================================

.. default-domain:: minio

.. contents:: Table of Contents
   :local:
   :depth: 2


The procedure on this page creates a new bucket replication rule for two-way
"active-active" synchronization of objects between MinIO buckets.

.. image:: /images/active-active-replication.svg
   :width: 600px
   :alt: Active-Active Replication synchronizes data between two remote clusters.
   :align: center

MinIO server-side replication supports at most *two* MinIO clusters. Both
clusters *must* run MinIO.

- To configure replication between arbitrary S3-compatible services, use
  :mc-cmd:`mc mirror`.

- To configure one-way "active-passive" replication between MinIO clusters,
  see :ref:`minio-bucket-replication-serverside-oneway`.

MinIO Active-Active replication is designed for synchronizing objects between
two MinIO clusters. MinIO does not support Active-Active replication between
more than two clusters (multi-site). Enterprises looking to implement multi-site
replication should consider leveraging `MinIO SUBNET
<https://min.io/pricing?ref=docs>`__ support to access the expertise, planning,
and engineering resources required for addressing that use case. 

.. seealso::

   - Use the :mc-cmd:`mc replicate edit` command to modify an existing
     replication rule.

   - Use the :mc-cmd-option:`mc replicate edit` command with the
     :mc-cmd-option:`--state "disable" <mc replicate edit state>` flag to
     disable an existing replication rule.

   - Use the :mc-cmd:`mc replicate rm` command to remove an existing replication
     rule.

.. _minio-bucket-replication-serverside-twoway-requirements:

Requirements
------------

Enable Versioning on Source and Destination Buckets
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

MinIO relies on the immutability protections provided by versioning to
synchronize objects between the source and replication target.

Use the :mc-cmd:`mc version enable` command to enable versioning on 
*both* the source and destination bucket before starting this procedure:

.. code-block:: shell
   :class: copyable

   mc version enable ALIAS/PATH

- Replace :mc-cmd:`ALIAS <mc version enable TARGET>` with the
  :mc:`alias <mc alias>` of the MinIO cluster.

- Replace :mc-cmd:`PATH <mc version enable TARGET>` with the bucket on which
  to enable versioning.

Install and Configure ``mc`` with Access to Both Clusters.
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This procedure uses :mc:`mc` for performing operations on both the source and
destination MinIO cluster. Install :mc:`mc` on a machine with network access to
both source and destination clusters. See the ``mc`` 
:ref:`Installation Quickstart <mc-install>` for instructions on downloading and
installing ``mc``.

Use the :mc:`mc alias` command to create an alias for both MinIO clusters.
Alias creation requires specifying an access key for a user on the cluster.
This user **must** have permission to create and manage users and policies
on the cluster. Specifically, ensure the user has *at minimum*:

- :policy-action:`admin:CreateUser`
- :policy-action:`admin:ListUsers`
- :policy-action:`admin:GetUser`
- :policy-action:`admin:CreatePolicy`
- :policy-action:`admin:GetPolicy`
- :policy-action:`admin:AttachUserOrGroupPolicy`

.. _minio-bucket-replication-serverside-twoway-permissions:

Required Permissions
~~~~~~~~~~~~~~~~~~~~

Bucket replication requires specific permissions on the source and
destination clusters to configure and enable replication rules. 

.. tabs::

   .. tab:: Replication Admin

      The following policy provides permissions for configuring and enabling
      replication on a cluster. 

      .. literalinclude:: /extra/examples/ReplicationAdminPolicy.json
         :class: copyable
         :language: json

      - The ``"EnableRemoteBucketConfiguration"`` statement grants permission
        for creating a remote target for supporting replication.

      - The ``"EnableReplicationRuleConfiguration"`` statement grants permission
        for creating replication rules on a bucket. The ``"arn:aws:s3:::*``
        resource applies the replication permissions to *any* bucket on the
        source cluster. You can restrict the user policy to specific buckets
        as-needed.

      Use the :mc-cmd:`mc admin policy add` to add this policy to *both*
      clusters. You can then create a user on both clusters using
      :mc-cmd:`mc admin user add` and associate the policy to those users
      with :mc-cmd:`mc admin policy set`.

   .. tab:: Replication Remote User

      The following policy provides permissions for enabling synchronization of
      replicated data *into* the cluster. Use the :mc-cmd:`mc admin policy add`
      to add this policy to *both* clusters.

      .. literalinclude:: /extra/examples/ReplicationRemoteUserPolicy.json
         :class: copyable
         :language: json

      - The ``"EnableReplicationOnBucket"`` statement grants permission for 
        a remote target to retrieve bucket-level configuration for supporting
        replication operations on *all* buckets in the MinIO cluster. To
        restrict the policy to specific buckets, specify those buckets as an
        element in the ``Resource`` array similar to
        ``"arn:aws:s3:::bucketName"``.

      - The ``"EnableReplicatingDataIntoBucket"`` statement grants permission
        for a remote target to synchronize data into *any* bucket in the MinIO
        cluster. To restrict the policy to specific buckets, specify those 
        buckets as an element in the ``Resource`` array similar to 
        ``"arn:aws:s3:::bucketName/*"``.

      Use the :mc-cmd:`mc admin policy add` to add this policy to *both*
      clusters. You can then create a user on both clusters using
      :mc-cmd:`mc admin user add` and associate the policy to those users
      with :mc-cmd:`mc admin policy set`.

MinIO strongly recommends creating users specifically for supporting 
bucket replication operations. See 
:mc:`mc admin user` and :mc:`mc admin policy` for more complete
documentation on adding users and policies to a MinIO cluster.

Considerations
--------------

Replication of Existing Objects
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Starting with :mc:`mc` :minio-git:`RELEASE.2021-06-13T17-48-22Z
<mc/releases/tag/RELEASE.2021-06-13T17-48-22Z>` and :mc:`minio`
:minio-git:`RELEASE.2021-06-07T21-40-51Z
<minio/releases/tag/RELEASE.2021-06-07T21-40-51Z>`, MinIO supports automatically
replicating existing objects in a bucket.

MinIO requires explicitly enabling replication of existing objects using the
:mc-cmd-option:`mc replicate add replicate` or
:mc-cmd-option:`mc replicate edit replicate` and including the 
``existing-objects`` replication feature flag. This procedure includes the
required flags for enabling replication of existing objects.

Replication of Delete Operations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

MinIO supports replicating delete operations onto the target bucket. 
Specifically, MinIO can replicate versioning
:s3-docs:`Delete Markers <versioning-workflows.html>` and the deletion
of specific versioned objects:

- For delete operations on an object, MinIO replication also creates the delete
  marker on the target bucket.

- For delete operations on versions of an object,
  MinIO replication also deletes those versions on the target bucket.

MinIO requires explicitly enabling replication of delete operations using the
:mc-cmd-option:`mc replicate add replicate` or 
:mc-cmd-option:`mc replicate edit replicate`. This procedure includes the
required flags for enabling replication of delete operations and delete markers.

Replication of Encrypted Objects
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

MinIO supports replicating objects encrypted with automatic 
Server-Side Encryption (SSE-S3). Both the source and destination buckets
*must* have automatic SSE-S3 enabled for MinIO to replicate an encrypted object.

As part of the replication process, MinIO *decrypts* the object on the source
bucket and transmits the unencrypted object. The destination MinIO cluster then
re-encrypts the object using the destination bucket SSE-S3 configuration. MinIO
*strongly recommends* :ref:`enabling TLS <minio-TLS>` on both source and
destination clusters to ensure the safety of objects during transmission.

MinIO does *not* support replicating client-side encrypted objects 
(SSE-C).

Replication of Object Retention (WORM, Legal Hold)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

MinIO supports replicating objects with retention settings, such as
:abbr:`WORM (Write-Once Read-Many)` object locking or legal holds. Both the
source and destination bucket *must* have object locking enabled for MinIO
to replicate objects with their associated retention settings.

Procedure
---------

1) Configure User Accounts and Policies for Replication
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This step creates multiple users and policies on both MinIO clusters for
supporting replication operations. You can skip this step if both
clusters already have users with the necessary
:ref:`permissions <minio-bucket-replication-serverside-twoway-permissions>`.

The following examples use ``Alpha`` and ``Baker`` as placeholder :mc:`aliases
<mc alias>` for each MinIO cluster. You should replace these values with the
appropriate aliases for the MinIO clusters on which you are configuring bucket
replication. These examples assume that the specified aliases have
the necessary permissions for creating policies and users on both clusters. See
:ref:`minio-users` and :ref:`MinIO Policy Based Access Control <minio-policy>` for more complete documentation on
MinIO users and policies respectively.

A\) Create Replication Administrators
   The following code creates policies and users for supporting configuring
   replication on the ``Alpha`` and ``Baker`` clusters. Replace the
   password ``LongRandomSecretKey`` with a long, random, and secure secret key 
   as per your organizations best practices for password generation.

   .. code-block:: shell
      :class: copyable

      wget -O - https://docs.min.io/minio/baremetal/examples/ReplicationAdminPolicy.json | \
      mc admin policy add Alpha ReplicationAdminPolicy /dev/stdin
      mc admin user add Alpha alphaReplicationAdmin LongRandomSecretKey
      mc admin policy set Alpha ReplicationAdminPolicy user=alphaReplicationAdmin
      
      wget -O - https://docs.min.io/minio/baremetal/examples/ReplicationAdminPolicy.json | \
      mc admin policy add Baker ReplicationAdminPolicy /dev/stdin
      mc admin user add Baker bakerReplicationAdmin LongRandomSecretKey
      mc admin policy set baker ReplicationAdminPolicy user=bakerReplicationAdmin

B\) Create Remote Replication User
   The following code creates policies and users for supporting synchronizing data
   to the ``Alpha`` and ``Baker`` clusters. Replace the password
   ``LongRandomSecretKey`` with a long, random, and secure secret key as per your
   organizations best practices for password generation.

   .. code-block:: shell
      :class: copyable
      
      wget -O - https://docs.min.io/minio/baremetal/examples/ReplicationRemoteUserPolicy.json | \
      mc admin policy add Alpha ReplicationRemoteUserPolicy /dev/stdin
      mc admin user add Alpha alphaReplicationRemoteUser LongRandomSecretKey
      mc admin policy set Alpha ReplicationRemoteUserPolicy user=alphaReplicationRemoteUser
      
      wget -O - https://docs.min.io/minio/baremetal/examples/ReplicationRemoteUserPolicy.json | \
      mc admin policy add Baker ReplicationRemoteUserPolicy /dev/stdin
      mc admin user add Baker bakerReplicationRemoteUser LongRandomSecretKey
      mc admin policy set Baker ReplicationRemoteUserPolicy user=bakerReplicationRemoteUser


2) Configure ``mc`` Access to the Remote Clusters
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Use the :mc-cmd:`mc alias set` command to add a replication-specific alias for
both remote clusters:

.. code-block:: shell
   :class: copyable

   mc alias set AlphaReplication HOSTNAME AlphaReplicationAdmin LongRandomSecretKey
   mc alias set BakerReplication HOSTNAME BakerReplicationAdmin LongRandomSecretKey

3) Create a Replication Target for Each Cluster
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Use the :mc-cmd:`mc admin bucket remote` command to create a replication target
for the destination cluster. For Active-Active replication, you must
issue this command on *both* clusters to create a remote in both directions

A\) Create a Replication Target for Alpha -> Baker
   The following command operates on the Alpha cluster to create a remote
   replication target on the Baker cluster:

   .. code-block:: shell
      :class: copyable

      mc admin bucket remote add AlphaReplication/SOURCEBUCKET \
         https://bakerReplicationRemoteUser:LongRandomSecretKey@HOSTNAME/DESTINATIONBUCKET \
         --service "replication" \
         [--sync]

   - Replace ``SOURCEBUCKET`` with the name of the source bucket on the 
     ``Alpha`` cluster.

   - Replace ``HOSTNAME`` with the URL of the ``Baker`` cluster.

   - Replace ``DESTINATIONBUCKET`` with the name of the target bucket on the
     ``Baker`` cluster.

   - Specify the :mc-cmd-option:`~mc admin bucket remote add sync` option to
     enable synchronous replication. Omit the option to use the default of 
     asynchronous replication. See the reference documentation for 
     :mc-cmd-option:`~mc admin bucket remote add sync` for more information
     on synchronous vs asynchronous replication.

   The command returns an ARN similar to the following. Copy this ARN for use in
   the following step. Note the ARN as
   associated to the ``Alpha`` cluster.

   .. code-block:: shell

      Role ARN = 'arn:minio:replication::<UUID>:DESTINATIONBUCKET'

B\) Create a Replication Target for Baker -> Alpha
   The following command operates on the Baker cluster to create a remote
   replication target on the Alpha cluster:

   .. code-block:: shell
      :class: copyable

      mc admin bucket remote add BakerReplication/SOURCEBUCKET \
         https://AlphaReplicationRemoteUser:LongRandomSecretKey@HOSTNAME/DESTINATIONBUCKET \
         --service "replication: \
         [--sync]

   - Replace ``SOURCEBUCKET`` with the name of the source bucket on the 
     ``Baker`` cluster.

   - Replace ``HOSTNAME`` with the URL of the ``Alpha`` cluster.

   - Replace ``DESTINATIONBUCKET`` with the name of the remote replication 
     target on the ``Alpha`` cluster.

   - Specify the :mc-cmd-option:`~mc admin bucket remote add sync` option to
     enable synchronous replication. Omit the option to use the default of 
     asynchronous replication. See the reference documentation for 
     :mc-cmd-option:`~mc admin bucket remote add sync` for more information
     on synchronous vs asynchronous replication.

   The command returns an ARN similar to the following. Copy this ARN for use in
   the following step. Note the ARN as
   associated to the ``Baker`` cluster.

   .. code-block:: shell

      Role ARN = 'arn:minio:replication::<UUID>:DESTINATIONBUCKET'


4) Create a New Bucket Replication Rule
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Use the :mc-cmd:`mc replicate add` command to add the new server-side
replication rule to the MinIO bucket. For Active-Active replication, you must
issue this command on *both* clusters to enable replication in both
directions.

A\) Create Replication Rule on Alpha

   The following command operates on the Alpha cluster to create a replication
   rule for synchronizing data to the Baker cluster. This command uses the ARN
   generated in the previous step:

   .. code-block:: shell
      :class: copyable

      mc replicate add AlphaReplication/SOURCEBUCKET \
         --remote-bucket DESTINATIONBUCKET \
         --arn 'arn:minio:replication::<UUID>:DESTINATIONBUCKET' \
         --replicate "delete,delete-marker,existing-objects"

   - Replace ``SOURCEBUCKET`` with the name of the bucket from which Alpha
     replicates data. The name *must* match the bucket specified when
     creating the remote target in the previous step.

   - Replace the ``DESTINATIONBUCKET`` with the name of the ``Baker`` bucket to
     which Alpha replicates data. The name *must* match the bucket specified
     when creating the remote target in the previous step.

   - Replace the ``--arn`` value with the ARN returned in the previous step. 
     Ensure you specify the ARN created on the ``Alpha`` cluster. You can use
     :mc-cmd:`mc admin bucket remote ls` to list all remote ARNs configured
     on the cluster.
   
   - The ``--replicate "delete,delete-marker,existing-objects"`` flag enables
     the following replication features:
      
     - :ref:`Replication of Deletes <minio-replication-behavior-delete>` 
     - :ref:`Replication of existing Objects <minio-replication-behavior-existing-objects>`
      
     See :mc-cmd-option:`mc replicate add replicate` for more complete
     documentation. Omit these fields to disable replication of delete operations
     or replication of existing objects respectively.

   Specify any other supported optional arguments for 
   :mc-cmd:`mc replicate add`.

B\) Create Replication Rule on Baker

   The following command operates on the Baker cluster to create a replication
   rule for synchronizing data to the Alpha cluster. This command uses the ARN
   generated in the previous step:

   .. code-block:: shell
      :class: copyable

      mc replicate add BakerReplication/SOURCEBUCKET \
         --remote-bucket DESTINATIONBUCKET \
         --arn 'arn:minio:replication::<UUID>:DESTINATIONBUCKET' \
         --replicate "delete,delete-marker,existing-objects"

   - Replace ``SOURCEBUCKET`` with the name of the bucket from which Baker
     replicates data. The name *must* match the bucket specified when
     creating the remote target in the previous step.

   - Replace the ``DESTINATIONBUCKET`` with the name of the ``Alpha`` bucket to
     which Baker replicates data. The name *must* match the bucket specified
     when creating the remote target in the previous step.

   - Replace the ``--arn`` value with the ARN returned in the previous step. 
     Ensure you specify the ARN created on the ``Alpha`` cluster. You can use
     :mc-cmd:`mc admin bucket remote ls` to list all remote ARNs configured
     on the cluster.

   - The ``--replicate "delete,delete-marker,existing-objects"`` flag enables
     the following replication features:
      
     - :ref:`Replication of Deletes <minio-replication-behavior-delete>` 
     - :ref:`Replication of existing Objects <minio-replication-behavior-existing-objects>`
      
     See :mc-cmd-option:`mc replicate add replicate` for more complete
     documentation. Omit these fields to disable replication of delete operations
     or replication fof existing objects respectively.

   Specify any other supported optional arguments for 
   :mc-cmd:`mc replicate add`.

   Specify any other supported optional arguments for 
   :mc-cmd:`mc replicate add`.

5) Validate the Replication Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Use :mc-cmd:`mc cp` to copy a new object to the ``Alpha`` source bucket. 

.. code-block:: shell
   :class: copyable

   mc cp ~/foo.txt Alpha/SOURCEBUCKET

Use :mc-cmd:`mc ls` to verify the object exists on the destination bucket:

.. code-block:: shell
   :class: copyable

   mc ls Baker/DESTINATIONBUCKET

Repeat this test by copying a new object to the ``Baker`` source bucket.

.. code-block:: shell
   :class: copyable

   mc cp ~/otherfoo.txt Baker/SOURCEBUCKET

Use :mc-cmd:`mc ls` to verify the object exists on the destination bucket:

.. code-block:: shell
   :class: copyable

   mc ls Alpha/DESTINATIONBUCKET

If the remote target was configured *without* the 
:mc-cmd-option:`~mc admin bucket remote add sync` option, the destination
bucket may have some delay before it receives the new object.