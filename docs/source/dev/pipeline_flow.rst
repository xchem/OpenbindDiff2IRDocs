Pipeline Flow and Triggers
==========================

Source: ``python-dlstbx/src/dlstbx/services/trigger_xchem.py`` (``DLSTriggerXChem``),
listening on the ``trigger_xchem`` queue.

.. _dev-chain:

Chain of processing
-------------------

::

     xia2 / autoPROC / xia2.multiplex   (upstream integration+scaling)
                  |
                dimple                  ispyb-postprocessing-dimple.json
                  |
                  v
     trigger_xchem  target=modelbuilding
       - waits for sibling upstream jobs (exponential backoff)
       - picks the 'best' dimple result
       - stages model_building/<dtag>/
                  | upserts a ligand restraints job
                  v
     grade2 restraints               ispyb-postprocessing-grade2.json
       wrapper: ligandrestraints
                  |
                  v
     trigger_xchem  target=hitidentification
       - records the dcid, decides per-dataset vs array, gates PanDDA2
                  | upserts "PanDDA2"/"Pipedream" (or "-array" variants)
                  v
     PanDDA2                          Pipedream
     ispyb-postprocessing-pandda2*    ispyb-postprocessing-pipedream*
     wrapper: pandda_xchem            wrapper: pipedream_xchem
                  |                        |
                  +----------+-------------+
                             | (non-array recipes only)
                             v
     trigger_xchem  target=xchem_collate
       - waits for the whole batch to finish (backoff)
                  | upserts "XChem-Collate"
                  v
     XChemCollate                     ispyb-postprocessing-xchem-collate.json
     wrapper: xchem_collate           -> also emails the visit team leader

The **array** recipes (``*-array.json``) deliberately have no collate step
currently. One slurm array is a single AutoProcProgram, so the "wait for
everything to finish" gate can never hold. Run collate yourself once the array has drained if using
array recipe when reprocessing visit data.

Target 1: ``modelbuilding``
---------------------------

``ModelBuildingParameters`` — ``dcid``, ``scaling_id`` (list),
``comparator_threshold`` (default 350), ``pipedream``, ``overwrite``,
``bulk_array``, ``timeout-minutes``, ``backoff-delay`` / ``backoff-max-try`` /
``backoff-multiplier``.

What it does, in order:

1. **Proposal filter.** Hardcoded ``ALLOWED_PROPOSALS`` list in the service
   (``lb42888``, ``sw44043``, ``sw44107``, ``lb36049``), plus ``PROPOSAL_ALIASES``
   (``mx41448`` → ``lb42888``) when it has been requested that data collected under a 
   particular beamline visit should end up in a particular labxchem visit . Anything else exits quietly.
2. **Recollection check.** If this dcid is not the latest for its dtag, exit —
   a recollection is presumed underway.
3. **Locate the labxchem visit** under ``/dls/labxchem/data/<proposal>`` by
   matching protein acronym / puck code / puck position / dtag against each
   candidate visit's soakDB (``find_xchem_visit_dir``). Exits if none matches.
4. **Wait for siblings.** Exits if a ``dimple`` job is already in
   flight for this dcid; checkpoints with exponential backoff while any of
   ``xia2 dials``, ``xia2 3dii``, ``autoPROC``, ``autoPROC+STARANISO``,
   ``xia2.multiplex`` are still running. ``fast_dp`` upstream → exit.
5. **Choose the dataset.** Rank by ``meanIOverSigI × completeness ×
   nTotalUniqueObservations``, preferring the user-defined spacegroup from
   ``Crystal.spaceGroup``, restricted to the most recent dimple batch (a new batch
   starts wherever there is a ≥12 h gap between dimple jobs).
6. **Read soakDB** for ``LibraryName`` / ``CompoundSMILES`` / ``CompoundCode``.
   Exits for DMSO solvent screens.
7. **Stage the dataset** (see layout below) and write the ``.smiles`` file.
   Without ``overwrite``, an existing ``<dtag>/compound/`` directory means exit —
   this is the idempotency guard.
8. **Fire one ligand restraints job** per dcid. Grade2 used by default, ACEDRG for industrial codes.

Target 2: ``hitidentification``
-------------------------------

``HitIndentificationParameters`` — ``dcid``, ``xchem_visit_dir`` (**required**,
this target does not re-derive it), ``scaling_id``, ``comparator_threshold``
(default 300), ``pipedream``, ``overwrite``, ``bulk_array``.

Two modes:

``bulk_array: true``
   List every directory under
   ``model_building/``, write the sorted list to
   ``model_building/.bulk_array.json``, and fire one ``PanDDA2-array`` job (plus
   ``Pipedream-array`` if ``pipedream``) with ``n_datasets`` = the list length.
   The array index (``SLURM_ARRAY_TASK_ID``) maps into that JSON file, so **the
   file is the task→dtag mapping**.

``bulk_array: false``
   Per-dataset, the production path. Append ``{dcid: dtag}`` to
   ``model_building/.batch_dcids.json``, then:

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Recorded dcid count
     - Behaviour
   * - ``< comparator_threshold``
     - Pipedream fires; PanDDA2 skipped (not enough comparator datasets yet)
   * - ``== comparator_threshold``
     - Pipedream fires; **one PanDDA2 job per recorded dcid** — the backlog is
       flushed
   * - ``> comparator_threshold``
     - Pipedream fires; a single PanDDA2 for the current dtag

Pipedream fires on *every* call regardless — it needs no comparators.
The count comes from the JSON file, not from ISPyB, so deleting
``.batch_dcids.json`` resets the gate and will re-flush the whole backlog at the
next threshold crossing.

Target 3: ``xchem_collate``
---------------------------

``CollateParameters`` — ``dcid``, ``program_id`` (the AutoProcProgramId of the job
that triggered it), ``scaling_id``, ``processing_directory``, ``pipedream``,
``overwrite``, backoff settings.

1. Gather every dcid for this target — matched by ``Protein.acronym`` within the
   proposal, not by visit.
2. **Abort** if any ``xia2 dials`` / ``PanDDA2`` / ``Pipedream`` program with an
   ``autoProcProgramId > program_id`` exists: a newer job has started and will
   trigger its own collate.
3. **Checkpoint with backoff** while any of those programs are still running
   (started within the last 8 h).
4. Exit if another ``XChemCollate`` job was launched for this target in the last
   180 minutes.
5. Fire one ``XChem-Collate`` job, keyed on ``max(dcids)``, carrying the team
   leader's email for the notification step.


On-disk layout
--------------

Everything the automated pipeline writes lives under ``auto/``, keeping it
separate from the manual XChemExplorer tree::

    /dls/labxchem/data/<proposal>/<visit>/
    ├── .user.yaml                              # configuration file with per-visit settings; autoprocessing.pandda -> PanDDA2 args
    └── processed/         
        └── auto/
            ├── database/autosoakDBDataFile.sqlite   # our writable copy
            ├── xchemalign/                          # autoXCA collator/aligner output
            └── analysis/
                ├── model_building/
                │   ├── .batch_dcids.json            # {dcid: dtag} gating map
                │   ├── .bulk_array.json             # ordered dtag list for array jobs
                │   └── <dtag>/
                │       ├── dimple.pdb, dimple.mtz
                │       ├── <dtag>.free.mtz          # the upstream scaled mtz
                │       ├── restraints.log
                │       └── compound/
                │           ├── <code>.smiles        # written by modelbuilding
                │           ├── <code>.cif           # written by grade2
                │           └── <code>.pdb
                ├── pandda2/
                │   ├── panddas/processed_datasets/<dtag>/
                │   │   ├── pandda2.log, rhofit.log, events.yaml
                │   │   ├── best_score.txt
                │   │   ├── pandda2_results.json
                │   │   └── modelled_structures/<dtag>-pandda-model.pdb
                │   └── score_<lo>-<hi>/              # symlink buckets from collate
                └── pipedream/
                    ├── <dtag>/postrefine-<code>/refine.pdb, refine.mtz
                    ├── Pipedream_output.json
                    └── Pipedream_results/

Gating files, and what deleting them does
-----------------------------------------

.. list-table::
   :header-rows: 1
   :widths: 30 20 50

   * - File
     - Written by
     - Effect of deleting
   * - ``model_building/<dtag>/compound/``
     - ``modelbuilding``
     - Re-staging becomes possible without ``overwrite: true``
   * - ``model_building/.batch_dcids.json``
     - ``hitidentification``
     - Resets the comparator count; the whole backlog re-fires at the next
       threshold crossing

``overwrite`` semantics
-----------------------

* **modelbuilding**: ``mkdir(exist_ok=overwrite)`` — re-stage over an existing
  ``model_building/<dtag>``.
* **pandda_xchem**: ``shutil.rmtree(processed_datasets/<dtag>)`` before running.
