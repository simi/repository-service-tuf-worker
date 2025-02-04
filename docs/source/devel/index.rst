###########
Development
###########


.. include:: design.rst



Component level
===============

.. image:: /_static/repository-service-tuf-worker-C3.png


Component Specific
==================

Distributed Asynchronous Signing
--------------------------------

This describes the Distributed Asynchronous Signing with other specific TUF
Metadata processes.


Bootstrap
.........

* if the included root has enough signatures, task is finalized right away
* otherwise, task is put in pending state and half-signed root is cached
  (RSTUF Setting: ``ROOT_SIGNING``)

.. note::

   See ``BOOTSTRAP`` and ``ROOT_SIGNING`` states reference in
   `Architecture Design: TUF Repository Settings
   <https://repository-service-tuf.readthedocs.io/en/stable/devel/design.html#tuf-repository-settings>`_.

.. uml::

   @startuml
      partition "Bootstrap"{
         start
         if (Bootstrap is "<task-id>") then (True)
            end
         elseif (Bootstrap is "pre-<task-id>") then (True)
            end
         elseif (Bootstrap is "signing-<task-id>") then (True)
            end
         else
            : Bootstrap is "None";
         endif
      }

      partition "Threshold" {
         if (Threshold is complete) then (True)
            : Finish Bootstrap;
            : "BOOTSTRAP" set to "<task-id>";

         else (False)
            : "ROOT_SIGNING" set to Root Metadata;
            : "BOOTSTRAP" set to "signing-<task-id>";
         endif
      }
      stop

    @enduml

Adding/Removing targets
-----------------------

As mentioned at the container level, the domain of ``repository-service-tuf-worker``
(Repository Worker) is managing the TUF Repository Metadata.
The Repository Worker has an Metadata Repository (`MetadataRepository
<repository_service_tuf_worker.html#repository_service_tuf_worker.repository.MetadataRepository>`_) implementation
using `python-tuf <https://theupdateframework.readthedocs.io/en/latest/>`_.

The repository implementation has different methods such as adding new targets,
removing targets, bumping role metadata versions (ex: Snapshot and Timestamp),
etc.

The Repository Worker handles everything as a task.
To handle the tasks, the Repository Worker uses `Celery
<https://docs.celeryq.dev/en/stable/>`_ as Task Manager.

We have two types of tasks:

- First are tasks that Repository Work consumes from the Broker Server are
  tasks published by the `Repository Service for TUF API
  <https://github.com/repository-service-tuf/repository-service-tuf-api>`_ in the ``repository_metadata``
  queue, sent by an API User.
- Second are tasks that Repository Work generates in the queue
  ``rstuf_internals``. Those are internal tasks for the Repository Worker
  maintenance.

The tasks are defined in the ``repository-service-tuf-worker/app.py```, and uses `Celery
Beat <https://docs.celeryq.dev/en/stable/userguide/periodic-tasks.html>`_ as
scheduler.

The repository Worker has two maintenance tasks:

- **Bump Roles** that contain online keys ("Snapshot", "Timestamp" and Hahsed
  Bins ("bins-XX").
- **Publish the new Hashed Bins Target Roles** ("bins-XX") with new/removed
  targets.

About **Bump Roles** (``bump_online_roles``) that contain online keys is easy.
These roles have short expiration (defined during repository configuration) and
must be "bumped" frequently. The implementation in the RepositoryMetadata

**Publish the new Hashed Bins Target Roles** (``publish_targets``) is part of the
solution for the :ref:`Repository Worker scalability, Issue 17
<devel/known_issues:(Solved) Scalability>`.

To understand more, every time the API sends a task to add a new target, the
Hashed Bins Roles must be changed to add the new target(s), followed by a new
Snapshot and Timestamp versions.

.. uml::

  @startuml
      !pragma useVerticalIf
      start
      :Add/Remove target(s);
      :1. Add the target(s) to the Hashed Bin Role;
      :2. Generate a new version;
      :2. Persist the new Hashed Bin Role in the Storage;
      :4. Update Hashed Bin Role version in the Snapshot meta;
      :5. Bump Snapshot version;
      :6. Persist the new Snapshot in the Storage;
      :7. Update Snapshot Version in the Timestamp;
      :8. Bump Timestamp Version;
      :9. Persist the new Timestamp in the Storage;
      stop
    @enduml

To give more flexibility to multiple Repository Workers to handle multiple
tasks and not wait until the entiry flow is done, per each task, we split it.

We use the 'waiting time' to alternate between tasks.

.. note::

   This is valid flow for the Repository Metadata Methods `add_targets
   <repository_service_tuf_worker.html#repository_service_tuf_worker.repository.MetadataRepository.add_targets>`_
   and `remove_targets
   <repository_service_tuf_worker.html#repository_service_tuf_worker.repository.MetadataRepository.remove_targets>`_

Repository Worker adds/removes the target to the SQL Database.

It means the multiple Repository Workers can write multiple Targets
(``TargetFiles``) simultaneously from various tasks in the Database.

The **Publish the new Hashed Bins Target Roles** is a synchronization between
the SQL Database and the Hashed Bins Target Roles in the Backend Storage (i.e.
JSON files in the filestytem)

When a task finishes, it sends a task the **Publish the new Hashed Bins Target
Roles** .

Every minute, the routine task **Publish the new Hashed Bins Target Roles**
also runs.

The task will continue running and waiting until all the targets are persisted to the
Repository Metadata backend.

The **Publish the new Hashed Bins Target Roles** task runs once per time to using
locks [#f1]_ . It will  will do:

.. uml::

   @startuml
      partition "publish targets" {
         start
         repeat
         repeat while (Try LOCK publish_targets) is (Waiting)
         if (delegated role has NO new targets files) then (True)
            stop
         else
            :Query all delegated role with target files changed;
            repeat :For each delegated role;
               :Clean the the role delegated metadata target files;
               :Add the target files with 'action' ADD;
               :Bump delegated role version;
            repeat while (Persist the delegated role new version)
            :Bump Snapshot Version with new targets;
            :Bump Timestamp Version with new Snapshot version;
            :Persist the new Timestamp in the Storage and Update the SQL;
            :UNLOCK publish_targets;
         stop
         endif
      }
    @enduml


.. [#f1]

   Lock is used Celery task. `It is used to ensure a task is only executed one
   at a time <https://docs.celeryq.dev/en/stable/tutorials/task-cookbook.html
   ?highlight=Task%20cookbook#ensuring-a-task-is-only-executed-one-at-a-time>`_
   . I avoid that two tasks write the same metadata, causing a race condition.

Important issues/problems
#########################

.. toctree::
   :maxdepth: 1

   known_issues

Implementation
##############

.. toctree::
   :maxdepth: 2

   repository_service_tuf_worker
   repository_service_tuf_worker.models
   repository_service_tuf_worker.models.targets
   repository_service_tuf_worker.services
   repository_service_tuf_worker.services.storage
   repository_service_tuf_worker.services.keyvault
   modules