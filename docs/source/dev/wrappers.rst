The Wrappers
============

Wrapper names (``--wrap``) come from ``known_wrappers`` in
``python-dlstbx/setup.py``: ``ligandrestraints``, ``pandda_xchem``,
``pipedream_xchem``, ``xchem_collate``, ``dimple``, ``ligand_fit``.

Running a wrapper directly
--------------------------

Every recipe's cluster step is just:

.. code-block:: bash

   dlstbx.wrap --wrap <wrapper_name> --recipewrapper "$RECIPEWRAP"

So you can re-run a failed job by hand from an existing ``.recipewrap``:

.. code-block:: bash

   module load dials/latest
   module load buster/20260424
   dlstbx.wrap --wrap pandda_xchem --recipewrapper /path/to/pandda/.recipewrap

``ligandrestraints`` — ``ligandrestraints.py``
----------------------------------------------

Reads the single ``compound/<code>.smiles`` staged by ``modelbuilding`` and runs:

.. code-block:: bash

   grade2 --in <code>.smiles --itype smi --out <code> -f > restraints.log

then renames ``<code>.restraints.cif`` → ``<code>.cif`` and ``<code>.xyz.pdb`` →
``<code>.pdb``. One hour timeout. Zero or multiple ``.smiles`` files is a hard
failure. Both PanDDA2 and Pipedream depend on that ``.cif`` existing.

``pandda_xchem`` — ``pandda_xchem.py``
--------------------------------------

``job_parameters``: ``dcid``, ``xchem_visit_dir``, ``processing_directory``,
``dtag``, ``n_datasets``, ``scaling_id``, ``database_path``, ``overwrite``,
``timeout-minutes``.

If ``n_datasets > 1`` it's an array task: ``dtag`` comes from
``model_building/.bulk_array.json[SLURM_ARRAY_TASK_ID - 1]`` and ISPyB
attachments are suppressed (``batch=True``).

Stages:

1. **PanDDA2** — ``process_dataset.py --data_dirs=<model_building>
   --out_dir=<panddas> --dtag=<dtag> --use_ligand_data=True --local_cpus=4``,
   plus any per-visit arguments from ``<visit>/.user.yaml``:

   .. code-block:: yaml

      autoprocessing:
        pandda:
          some_key: some_value    # becomes --some_key=some_value

   With ``overwrite``, ``processed_datasets/<dtag>`` is removed first.
2. **Event selection** — reads ``events.yaml``, takes the single highest-``Score``
   event. More than one binder per dataset is rare and the score ranks well.
3. **Map/model restriction before Rhofit** — the event map is masked to a
   radius around the event centroid (``mask_map``) and atoms within 10 Å of the
   centroid are stripped from the input model (``remove_nearby_atoms``). Both
   matter: Rhofit hunts non-binding-site density otherwise, and it masks the
   protein before building, so a model clipping the event makes autobuilding
   impossible.
4. **Rhofit** — ``pandda_rhofit.sh`` with a sigma-derived cut level.
5. **Scoring** — ``ligand_score.py`` per build against the z-map; PanDDA2's own
   internal autobuild is included in the comparison, scored with the event score.
   The winner is written to ``best_score.txt`` (which is what collate buckets on).
6. **Merge** — best ligand merged into the protein model as
   ``modelled_structures/<dtag>-pandda-model.pdb``; any previous model is kept as
   ``pandda-internal-fitted.pdb``; waters clashing with the ligand are removed.
7. **Output** — cropped event/z maps, an MVS html viewer, ``pandda2_results.json``.

Failure at stages 1–2 returns ``False`` after uploading whatever attachments
exist; a Rhofit failure is logged but does **not** abort (PanDDA2's internal
autobuild may still carry the dataset).

``pipedream_xchem`` — ``pipedream_xchem.py``
--------------------------------------------

``job_parameters``: ``dcid``, ``processing_directory``, ``dtag``, ``n_datasets``,
``scaling_id``, ``database_path``, ``timeout-minutes``. Same array handling as
above.

Inputs from ``model_building/<dtag>``: ``dimple.pdb``, ``dimple.mtz``,
``<dtag>.free.mtz``, ``compound/<code>.cif``. ``process_pdb_file()`` strips
crystallisation components from the dimple pdb first.

.. code-block:: text

   pipedream -nolmr -hklin <dtag>.free.mtz -xyzin dimple.pdb -hklref dimple.mtz
             -d <pipedream>/<dtag>
             -mrefine TLSbasic,WaterUpdatePkmaps -keepwater -remediate
             -sidechainrebuild -runpepflip -rhocommands -xclusters -nochirals
             -rhofit <code>.cif

BUSTER build is pinned in the wrapper (``/dls_sw/apps/GPhL/BUSTER/20260424``).

Post-processing (each step degrades gracefully, returning ``True`` with whatever
was produced): ``gemmi sf2map`` for 2fofc/fofc maps → cropped map + MVS html →
``edstats`` over the resolution range read back from ``pipedream_summary.json``.

Outputs of interest: ``postrefine-<code>/refine.pdb`` / ``refine.mtz``,
``report-<code>/report.pdf``, ``summary.out``, ``edstats.out``,
``pipedream_summary.json``.

``xchem_collate`` — ``xchem_collate.py``
----------------------------------------

``job_parameters``: ``processing_directory``, ``pipedream``, ``overwrite``,
``timeout-minutes``. Runs once per visit, not per dataset.

1. **PanDDA2 postrun** — ``postrun.py --data_dirs=<model_building>
   --out_dir=<panddas> --use_ligand_data=False``, producing the events/sites csvs.
2. **soakDB re-integration**

   * ``prepare_auto_db(processing_dir)`` copies the master soakDB to
     ``auto/database/autosoakDBDataFile.sqlite`` if absent, then syncs any new
     columns and any new ``mainTable`` rows from master into the copy. **All
     writes go to the copy, never the master.**
   * ``updatable_crystals(db_copy, overwrite)`` returns the set of CrystalNames
     this run may write. It is the single gate — computed once, used both for
     building the update dicts and for the bulk write.
   * ``symlink_score_buckets()`` symlinks each updatable dataset into
     ``pandda2/score_0.6-0.8/``, ``score_0.8-0.9/``, ``score_0.9-1/`` by its
     ``best_score.txt``, mirroring the panddas layout (``processed_datasets/`` +
     a filtered ``analyses/pandda_analyse_events.csv``) so models can be browsed
     by ligand score. Earlier batches' symlinks are preserved.
   * ``update_xchem_database()`` performs model selection and bulk-writes into
     the copy's ``mainTable`` (see below).
3. **Pipedream collate** (if ``pipedream``) — ``collate_pipedream_results.py``
   from the xchem micromamba env over ``Pipedream_output.json``, then
   ``write_pipedream_parameters()``.
4. **XChemAlign** — if ``processed/analysis/xchemalign/{config,assemblies}.yaml``
   exist, copies them to ``auto/xchemalign/`` and runs ``xchemalign.collator``
   then ``xchemalign.aligner``.
5. **Cleanup** — ``cleanup_setvar_files()`` removes orphaned autoBUSTER setvar
   logs.

.. note::

   Every stage is individually wrapped in try/except: collate logs and continues
   rather than failing the job, so a ``success`` status does not mean every stage
   ran. Read the job log, not just the ISPyB status.

Model selection (``update_xchem_database``)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Per dataset, in ``model_building``, skipping anything not in ``updatable`` or
with no ``compound/*.cif``:

.. list-table::
   :header-rows: 1
   :widths: 32 14 24 30

   * - Condition
     - Selected
     - ``RefinementOutcome``
     - ``RefinementLigandConfidence``
   * - ``refine.pdb`` exists and RSCC ≥ 0.8
     - Pipedream
     - ``4 - CompChem ready``
     - ``4 - High Confidence``
   * - ``refine.pdb`` exists and 0.7 < RSCC < 0.8
     - Pipedream
     - ``3 - In Refinement``
     - ``2 - Correct ligand, weak density``
   * - No Pipedream model above threshold, PanDDA2 model exists
     - PanDDA2
     - ``2 - PANDDA model``
     - from ``pandda_ligand_confidence()``
   * - Neither
     - —
     - row still written, no model
     - —

The model is ``pipedream/<dtag>/postrefine-*/refine.pdb``; the RSCC is the best
value in ``rhofit-*/Hit_corr.log``, and selection needs it strictly above
``rscc_thresh = 0.7``. Only on the Pipedream branch are the full refinement /
validation statistics pulled from ``pipedream_summary.json`` and the files
exported into the dataset dir.

.. note::

   ``RefinementLigandCC`` is the **pre-refinement** omit-map Rhofit CC from
   ``Hit_corr.log``, not the post-refinement summary ligandcc — deliberately, to
   avoid model bias.

Relevant helper modules: ``dlstbx.util.soakdb``, ``dlstbx.util.pandda``,
``dlstbx.util.xchem_collate_helpers``, ``dlstbx.util.pipedream_xchem_helpers``,
``dlstbx.util.symlink``.
