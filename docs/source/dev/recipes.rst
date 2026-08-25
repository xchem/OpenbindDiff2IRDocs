The XChem Recipes
=================

The recipes live in the zocalo repository and are what turn a trigger message
into a cluster job. Each cluster step is a ``dlstbx.wrap --wrap <wrapper>``
invocation — see :doc:`wrappers`.

Recipe reference
----------------

``ispyb-postprocessing-dimple.json``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Runs dimple in ``/tmp/{guid}`` and rsyncs results back. On completion, triggers:

* step 10 (on **both** success and failure) → ``DLS TriggerXChem``
  ``target: modelbuilding``, ``comparator_threshold: 300``, ``pipedream: true``

``ispyb-postprocessing-grade2.json``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Program ``Grade2``, wrapper ``ligandrestraints``, 1 CPU / 1 h. On success (step 7)
→ ``trigger_xchem target: hitidentification``, forwarding ``xchem_visit_dir``,
``comparator_threshold``, ``pipedream``, ``overwrite`` and the backoff settings.
Note it does *not* forward ``bulk_array`` — the production per-dataset path is
implied.

``ispyb-postprocessing-pandda2.json``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Program ``PanDDA2``, wrapper ``pandda_xchem``, 4 CPU × 12 GB, 3 h,
``partition: mx_low,cs04r``, ``timeout-minutes: 145``. Loads ``dials/latest`` and
``buster/20260424``. Success **and** failure route to step 7 →
``trigger_xchem target: xchem_collate`` with
``program_id: $ispyb_autoprocprogram_id``.

``ispyb-postprocessing-pandda2-array.json``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Same, plus ``"array": "1-{ispyb_reprocessing_parameters[n_datasets]}%350"`` and
``n_datasets`` in ``job_parameters``. Time limit 2:30:00. **No collate step** —
see :ref:`dev-chain`.

``ispyb-postprocessing-pipedream.json``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Program ``Pipedream``, wrapper ``pipedream_xchem``, 4 CPU × 4 GB, 4 h,
``timeout-minutes: 235``. Success and failure both route to step 6 →
``trigger_xchem target: xchem_collate``.

``ispyb-postprocessing-pipedream-array.json``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Array variant, ``%150`` concurrency cap, no collate step.

``ispyb-postprocessing-xchem-collate.json``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Program ``XChemCollate``, wrapper ``xchem_collate``, 4 CPU, 3 h,
``timeout-minutes: 179``, ``overwrite`` **hardcoded false**. Adds an ``email``
output channel → ``DLS Mail Notifications`` on the ``mailnotification`` queue.

Running recipes by hand with ``zocalo.go``
------------------------------------------

The command
~~~~~~~~~~~

.. code-block:: bash

   module load dials
   zocalo.go [options] [dcid]

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - Flag
     - Meaning
   * - ``-f FILE``
     - Run the recipe contained in this **local file**. This is the one you want
       for testing — it bypasses ``/dls_sw/apps/zocalo/live/recipes`` entirely
   * - ``-r NAME``
     - Run a **named** recipe, i.e. ``NAME.json`` in the live recipes dir.
       Repeatable
   * - ``-n``
     - Trigger without giving a data collection ID on the command line. Use
       whenever your recipe hardcodes or substitutes the dcid itself
   * - ``-s KEY=VALUE``
     - Set a variable for recipe evaluation. Every ``{KEY}`` in the recipe JSON
       is replaced by ``VALUE``. Repeatable
   * - ``-a APSID``
     - Supply an autoProcScalingId for downstream processing recipes
   * - ``-p``
     - The positional argument is a **reprocessing** (ProcessingJob) ID, not a
       dcid
   * - ``-e ENV``
     - Zocalo environment — ``devrmq`` for the local dev instance, omit for
       production
   * - ``-v``
     - Print the raw message before sending
   * - ``--dry-run``
     - Check everything is in place but don't actually send

``-f`` and ``-r`` are the two ways in. ``-f`` is for anything you are iterating
on; ``-r`` is for firing a production recipe as-is.

Reprocessing with a local recipe
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This is the everyday form:

.. code-block:: bash

   module load dials
   zocalo.go -n -f /dls/science/users/qvu59474/projects/reprocessing/my-trigger.json \
             -s dcid=20217382 \
             -s pipedream=false

* ``-n`` because the dcid comes from ``-s``, not from the command line.
* ``-s dcid=...`` fills the ``"dcid": "{dcid}"`` placeholder in the recipe.
* ``-s pipedream=false`` fills ``"pipedream": "{pipedream}"``, so the same file
  can run PanDDA2-only or PanDDA2+Pipedream without editing it.

Anything in the recipe written as ``{name}`` is a substitution slot. Anything
written as a literal value is fixed — so recipes with hardcoded dcids
(``test-trigger-bulk.json``, ``test-trigger-collate.json``) need editing rather
than ``-s``.


Shape of a trigger recipe
~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: json

   {
     "1": [ "Some description" ],

     "1": { "service": "DLS TriggerXChem",
            "queue": "trigger_xchem",
            "parameters": { "target": "modelbuilding",
                            "dcid": "{dcid}",
                            "comparator_threshold": 300,
                            "scaling_id": ["21038860"],
                            "pipedream": "{pipedream}",
                            "comment": "PanDDA2 triggered by Rowan",
                            "automatic": false } },

     "start": [ [1, []] ]
   }

Points that bite:

* ``scaling_id`` must be a **list**, even when only one entry is used
  (``parameters.scaling_id[0]`` is what the service reads).
* ``target`` must exactly match a ``trigger_<target>`` method on the service
  (``modelbuilding``, ``hitidentification``, ``xchem_collate``), otherwise the
  message is nacked.
* ``automatic: false`` marks the resulting ProcessingJob as manually triggered.
* Parameters not on the relevant pydantic model are ignored, not rejected — a
  typo'd key fails silently. Missing required fields raise a
  ``ValidationError`` that shows up in the service log and nacks the message.

Local dev zocalo
----------------

``zocalo/start_zocalo.sh`` brings up a private RabbitMQ plus the services this
pipeline needs:

.. code-block:: bash

   #!/usr/bin/bash
   pkill -f rabbitmq
   module load dials/qvu59474
   module load rabbitmq/tmp
   sleep 15
   zocalo.service -e devrmq -s Dispatcher -v &
   zocalo.service -e devrmq -s DLSTriggerXChem -v &
   zocalo.service -e devrmq -s DLSTrigger -v &
   zocalo.service -e devrmq -s DLSCluster -v &
   zocalo.service -e devrmq -s DLSISPyB -v &

Then fire recipes at it with ``-e devrmq``:

Notes:

* ``dials/qvu59474`` is the module built from the local ``python-dlstbx``
  checkout — that's how service/wrapper code changes get picked up.
* Each service shows up as **two** processes (frontend + worker). That's normal,
  not a duplicate service.
* ``DLSCluster`` submits to real Slurm, so a local run still puts real jobs on
  the cluster and writes to real ``/dls/labxchem`` paths. There is no dry-run for
  that — point the recipe at a scratch visit if you don't want that.