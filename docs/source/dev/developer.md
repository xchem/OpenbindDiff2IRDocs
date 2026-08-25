# XChem pipeline flow

Source: `python-dlstbx/src/dlstbx/services/trigger_xchem.py` (`DLSTriggerXChem`),
listening on the `trigger_xchem` queue.

## Chain of processing

```
 xia2 / autoPROC / xia2.multiplex   (upstream integration+scaling)
              |
            dimple                  ispyb-postprocessing-dimple.json
              |
              v
 trigger_xchem  target=modelbuilding
   - waits for sibling upstream jobs (exponential backoff)
   - picks the best dimple result
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
                         | (non-array recipes only, step 7 / step 6)
                         v
 trigger_xchem  target=xchem_collate
   - waits for the whole batch to finish (backoff)
              | upserts "XChem-Collate"
              v
 XChemCollate                     ispyb-postprocessing-xchem-collate.json
 wrapper: xchem_collate           -> also emails the visit team leader
```

The **array** recipes (`*-array.json`) deliberately have no collate step currently. One slurm array is a single AutoProcProgram, so the "wait for everything to finish" gate can never hold; if collate were wired in it would fire once per array task. Run collate yourself once the array has drained if using array recipe when reprocessing.

## Target 1: `modelbuilding`

`ModelBuildingParameters` — `dcid`, `scaling_id` (list), `comparator_threshold`
(default 350), `pipedream`, `overwrite`, `bulk_array`, `timeout-minutes`,
`backoff-delay` / `backoff-max-try` / `backoff-multiplier`.

What it does, in order:

1. **Proposal filter.** Hardcoded `ALLOWED_PROPOSALS` list in the service
   (`lb42888`, `sw44043`, `sw44107`, `lb36049`), plus `PROPOSAL_ALIASES`
   (`mx41448` → `lb42888`). Anything else exits quietly.
2. **Recollection check.** If this dcid is not the latest for its dtag, exit —
   a recollection is presumed underway.
3. **Locate the labxchem visit** under `/dls/labxchem/data/<proposal>` by
   matching protein acronym / puck code / puck position / dtag against each
   candidate visit's soakDB (`find_xchem_visit_dir`). Exits if none matches.
4. **Wait for siblings.** Exits if a `dimple` or `PanDDA2` job is already in
   flight for this dcid; checkpoints with exponential backoff while any of
   `xia2 dials`, `xia2 3dii`, `autoPROC`, `autoPROC+STARANISO`, `xia2.multiplex`
   are still running. `fast_dp` upstream → exit.
5. **Choose the dataset.** Rank by `meanIOverSigI × completeness ×
   nTotalUniqueObservations`, preferring the user-defined spacegroup from
   `Crystal.spaceGroup`, restricted to the most recent dimple batch (a new batch
   starts wherever there is a ≥12 h gap between dimple jobs).
6. **Read soakDB** for `LibraryName` / `CompoundSMILES` / `CompoundCode`. Exits
   for DMSO solvent screens and for rows with no SMILES.
7. **Stage the dataset** (see layout below) and write the `.smiles` file.
   Without `overwrite`, an existing `<dtag>/compound/` directory means exit —
   this is the idempotency guard.
8. **Fire one Grade2 job** per dcid.

## Target 2: `hitidentification`

`HitIndentificationParameters` — `dcid`, `xchem_visit_dir` (**required**, this
target does not re-derive it), `scaling_id`, `comparator_threshold` (default
300), `pipedream`, `overwrite`, `bulk_array`.

Two modes:

**`bulk_array: true`** — ignore the batching logic entirely. List every
directory under `model_building/`, write the sorted list to
`model_building/.bulk_array.json`, and fire one `PanDDA2-array` job (plus
`Pipedream-array` if `pipedream`) with `n_datasets` = the list length. The array
index (`SLURM_ARRAY_TASK_ID`) maps into that JSON file, so **the file is the
task→dtag mapping** — don't regenerate it while an array is running.

**`bulk_array: false`** (per-dataset, the production path) — append
`{dcid: dtag}` to `model_building/.batch_dcids.json`, then:

| Recorded dcid count | Behaviour |
|---|---|
| `< comparator_threshold` | Pipedream fires; PanDDA2 skipped (not enough comparator datasets yet) |
| `== comparator_threshold` | Pipedream fires; **one PanDDA2 job per recorded dcid** — the backlog is flushed |
| `> comparator_threshold` | Pipedream fires; a single PanDDA2 for the current dtag |

Pipedream fires on *every* call regardless — it needs no comparators.
The count comes from the JSON file, not from ISPyB, so deleting
`.batch_dcids.json` resets the gate and will re-flush the whole backlog at the
next threshold crossing.

## Target 3: `xchem_collate`

`CollateParameters` — `dcid`, `program_id` (the AutoProcProgramId of the job that
triggered it), `scaling_id`, `processing_directory`, `pipedream`, `overwrite`,
backoff settings.

1. Gather every dcid for this target — matched by `Protein.acronym` within the
   proposal, not by visit.
2. **Abort** if any `xia2 dials` / `PanDDA2` / `Pipedream` program with an
   `autoProcProgramId > program_id` exists: a newer batch has started and will
   trigger its own collate. (This is why manual collate recipes use a huge
   `program_id`.)
3. **Checkpoint with backoff** while any of those programs are still running
   (started within the last 8 h).
4. Exit if another `XChemCollate` job was launched for this target in the last
   180 minutes.
5. Fire one `XChem-Collate` job, keyed on `max(dcids)`, carrying the team
   leader's email for the notification step.

> The team leader lookup is currently overridden — `team_leader_email` is
> hardcoded to `qvu59474@diamond.ac.uk` immediately after
> `get_visit_team_leader_email()` is called. Revert that line before going live.

## On-disk layout

Everything the automated pipeline writes lives under `auto/`, keeping it
separate from the manual XChemExplorer tree:

```
/dls/labxchem/data/<proposal>/<visit>/
├── .user.yaml                              # configuration file with per-visit settings; autoprocessing.pandda -> PanDDA2 args
└── processed/
    ├── database/soakDBDataFile.sqlite      # the master soakDB (read-only to us)
    ├── analysis/xchemalign/                # user-provided config.yaml + assemblies.yaml
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
```

## Gating files, and what deleting them does

| File | Written by | Effect of deleting |
|------|-----------|--------------------|
| `model_building/<dtag>/compound/` | `modelbuilding` | Re-staging becomes possible without `overwrite: true` |
| `model_building/.batch_dcids.json` | `hitidentification` | Resets the comparator count; the whole backlog re-fires at the next threshold crossing |
| `model_building/.bulk_array.json` | `hitidentification` (bulk) | Breaks any in-flight array — tasks resolve their dtag from this file at runtime |
| `auto/database/autosoakDBDataFile.sqlite` | collate | Rebuilt from the master on the next collate; loses any automated `RefinementOutcome` values, so previously-written rows become updatable again |

## `overwrite` semantics

- **`modelbuilding`**: `mkdir(exist_ok=overwrite)` — re-stage over an existing
  `model_building/<dtag>`.
- **`pandda_xchem`**: `shutil.rmtree(processed_datasets/<dtag>)` before running.


## The XChem recipes

### `ispyb-postprocessing-dimple.json`

Runs dimple in `/tmp/{guid}` and rsyncs results back. Two triggers hang off it:

- step 9 (on `ispyb` channel, after `insert_dimple_result`) → `DLS Trigger`
  `target: ligand_fit`
- step 10 (on **both** success and failure) → `DLS TriggerXChem`
  `target: modelbuilding`, `comparator_threshold: 300`, `pipedream: true`

Firing on failure too is deliberate — a failed dimple for one sweep shouldn't
stop the dataset being considered, since `modelbuilding` picks the best of
several dimple jobs anyway.

### `ispyb-postprocessing-grade2.json`

Program `Grade2`, wrapper `ligandrestraints`, 1 CPU / 1 h. On success (step 7)
→ `trigger_xchem target: hitidentification`, forwarding `xchem_visit_dir`,
`comparator_threshold`, `pipedream`, `overwrite` and the backoff settings.
Note it does *not* forward `bulk_array` — the production per-dataset path is
implied.

### `ispyb-postprocessing-pandda2.json`

Program `PanDDA2`, wrapper `pandda_xchem`, 4 CPU × 12 GB, 3 h,
`partition: mx_low,cs04r`, `timeout-minutes: 145`. Loads `dials/latest` and
`buster/20260424`. Success **and** failure route to step 7 →
`trigger_xchem target: xchem_collate` with
`program_id: $ispyb_autoprocprogram_id`.

### `ispyb-postprocessing-pandda2-array.json`

Same, plus `"array": "1-{ispyb_reprocessing_parameters[n_datasets]}%350"` and
`n_datasets` in `job_parameters`. Time limit 2:30:00. **No collate step** — see
[pipeline-flow.md](pipeline-flow.md#the-chain).

### `ispyb-postprocessing-pipedream.json`

Program `Pipedream`, wrapper `pipedream_xchem`, 4 CPU × 4 GB, 4 h,
`timeout-minutes: 235`. Success and failure both route to step 6 →
`trigger_xchem target: xchem_collate`.

### `ispyb-postprocessing-pipedream-array.json`

Array variant, `%150` concurrency cap, no collate step.

### `ispyb-postprocessing-xchem-collate.json`

Program `XChemCollate`, wrapper `xchem_collate`, 4 CPU, 3 h,
`timeout-minutes: 179`, `overwrite` **hardcoded false**. Adds an `email`
output channel → `DLS Mail Notifications` on the `mailnotification` queue.

# Running recipes by hand with `zocalo.go`

## The command

```bash
module load dials
zocalo.go [options] [dcid]
```

| Flag | Meaning |
|------|---------|
| `-f FILE` | Run the recipe contained in this **local file**. This is the one you want for testing — it bypasses `/dls_sw/apps/zocalo/live/recipes` entirely |
| `-r NAME` | Run a **named** recipe, i.e. `NAME.json` in the live recipes dir. Repeatable |
| `-n` | Trigger without giving a data collection ID on the command line. Use whenever your recipe hardcodes or substitutes the dcid itself |
| `-s KEY=VALUE` | Set a variable for recipe evaluation. Every `{KEY}` in the recipe JSON is replaced by `VALUE`. Repeatable |
| `-a APSID` | Supply an autoProcScalingId for downstream processing recipes |
| `-p` | The positional argument is a **reprocessing** (ProcessingJob) ID, not a dcid |
| `-e ENV` | Zocalo environment — `devrmq` for the local dev instance, omit for production |
| `-v` | Print the raw message before sending |
| `--dry-run` | Check everything is in place but don't actually send |

`-f` and `-r` are the two ways in. `-f` is for anything you are iterating on;
`-r` is for firing a production recipe as-is.

## Reprocessing with a local recipe

This is the everyday form:

```bash
module load dials
zocalo.go -n -f /dls/science/users/qvu59474/projects/reprocessing/test-trigger.json \
          -s dcid=20217382 \
          -s pipedream=false
```

- `-n` because the dcid comes from `-s`, not from the command line.
- `-s dcid=...` fills the `"dcid": "{dcid}"` placeholder in the recipe.
- `-s pipedream=false` fills `"pipedream": "{pipedream}"`, so the same file can
  run PanDDA2-only or PanDDA2+Pipedream without editing it.

Anything in the recipe written as `{name}` is a substitution slot. Anything
written as a literal value is fixed — so recipes with hardcoded dcids
(`test-trigger-bulk.json`, `test-trigger-collate.json`) need editing rather than
`-s`.

## The test-trigger recipes

Working copies live in two places — `zocalo/recipes/` (committed) and
`/dls/science/users/qvu59474/projects/reprocessing/` (scratch). They are all
one-step recipes that just post a message to a trigger service.

| File | Target | What it does |
|------|--------|--------------|
| `test-trigger.json` | `modelbuilding` | Full chain from a dcid: pick the best dimple, stage `model_building/<dtag>`, fire grade2 → hitidentification → PanDDA2/Pipedream → collate. `{dcid}` and `{pipedream}` are substitutable |
| `test-trigger-pipedream.json` | `hitidentification` | Pipedream-only for one already-staged dataset (`pandda: false`) |
| `test-trigger-bulk.json` | `hitidentification` | `bulk_array: true` — one Slurm array job over every dtag in `model_building` |
| `test-trigger-legacy.json` | `hitidentification` | Legacy-tree staging + bulk array. **Note** `use_existing_modeldir` in this file is not a field on `HitIndentificationParameters` and is silently ignored |
| `test-trigger-collate.json` | `xchem_collate` | Collate only. Uses a deliberately huge `program_id` (`999999999`) so the "is a newer job running?" gate passes |
| `test-trigger-dimple2.json` | `dimple` (DLSTrigger) | Fire dimple directly |
| `test-trigger-lf.json` | `ligand_fit` (DLSTrigger) | Fire ligand_fit directly |

### Shape of a trigger recipe

```json
{
  "1": [ "Free-text description, ignored" ],

  "1": { "service": "DLS TriggerXChem",
         "queue": "trigger_xchem",
         "parameters": { "target": "modelbuilding",
                         "dcid": "{dcid}",
                         "comparator_threshold": 100,
                         "scaling_id": ["21038860"],
                         "pipedream": "{pipedream}",
                         "comment": "PanDDA2 triggered by Rowan",
                         "automatic": false } },

  "start": [ [1, []] ]
}
```

Points that bite:

- The duplicate `"1"` key is intentional in these files — JSON keeps the **last**
  one, so the description block is discarded. Don't "fix" it by renumbering, but
  do be aware your description is not being read.
- `scaling_id` must be a **list**, even when only one entry is used
  (`parameters.scaling_id[0]` is what the service reads).
- `target` must exactly match a `trigger_<target>` method on the service
  (`modelbuilding`, `hitidentification`, `xchem_collate`), otherwise the message
  is nacked.
- `automatic: false` marks the resulting ProcessingJob as manually triggered.
- Parameters not on the relevant pydantic model are ignored, not rejected — a
  typo'd key fails silently. Missing required fields raise a `ValidationError`
  that shows up in the service log and nacks the message.

## Local dev zocalo

`zocalo/start_zocalo.sh` brings up a private RabbitMQ plus the services this
pipeline needs:

```bash
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
```

Then fire recipes at it with `-e devrmq`:

```bash
zocalo.go -e devrmq -n -f recipes/test-trigger.json -s dcid=20217382 -s pipedream=false
```

Notes:

- `dials/qvu59474` is the module built from the local `python-dlstbx` checkout —
  that's how service/wrapper code changes get picked up.
- Each service shows up as **two** processes (frontend + worker). That's normal,
  not a duplicate service.
- `Dispatcher` is required: it is what turns a recipe message into the per-step
  messages the other services consume.
- `DLSCluster` submits to real Slurm, so a local run still puts real jobs on the
  farm and writes to real `/dls/labxchem` paths. There is no dry-run for that —
  point the recipe at a scratch visit if you don't want that.

## Checking what happened

```bash
# Service logs: whatever terminal start_zocalo.sh was run from (-v = verbose)

# Slurm, from a node that has the scheduler configured
ssh wilson 'squeue -u $USER'
ssh wilson 'sacct -u $USER -S now-2hours --format=JobID,JobName%20,State,ExitCode,Elapsed'

# Per-job launch dir and recipewrapper, as named in the recipe:
#   <ispyb_working_directory>/<step>/.recipewrap
#   <ispyb_working_directory>/<step>/.launch
```

The `.recipewrap` file is the full serialised recipe the wrapper received —
reading it is the quickest way to see exactly which `job_parameters` a wrapper
was handed, including the ISPyB substitutions.

## Running a wrapper directly

Every recipe's cluster step is just:

```bash
dlstbx.wrap --wrap <wrapper_name> --recipewrapper "$RECIPEWRAP"
```

So you can re-run a failed job by hand from an existing `.recipewrap`:

```bash
module load dials/latest
module load buster/20260424
dlstbx.wrap --wrap pandda_xchem --recipewrapper /path/to/pandda/.recipewrap
```

## Wrappers
Wrapper names (`--wrap`) come from `known_wrappers` in
`python-dlstbx/setup.py`: `ligandrestraints`, `pandda_xchem`, `pipedream_xchem`,
`xchem_collate`, `dimple`, `ligand_fit`.

## `ligandrestraints` — `ligandrestraints.py`

Reads the single `compound/<code>.smiles` staged by `modelbuilding` and runs:

```
grade2 --in <code>.smiles --itype smi --out <code> -f > restraints.log
```

then renames `<code>.restraints.cif` → `<code>.cif` and `<code>.xyz.pdb` →
`<code>.pdb`. One hour timeout. Zero or multiple `.smiles` files is a hard
failure. Both PanDDA2 and Pipedream depend on that `.cif` existing.

## `pandda_xchem` — `pandda_xchem.py`

`job_parameters`: `dcid`, `xchem_visit_dir`, `processing_directory`, `dtag`,
`n_datasets`, `scaling_id`, `database_path`, `overwrite`, `timeout-minutes`.

If `n_datasets > 1` it's an array task: `dtag` comes from
`model_building/.bulk_array.json[SLURM_ARRAY_TASK_ID - 1]` and ISPyB attachments
are suppressed (`batch=True`) — SynchWeb can't represent per-task attachments.

Stages:

1. **PanDDA2** — `process_dataset.py --data_dirs=<model_building>
   --out_dir=<panddas> --dtag=<dtag> --use_ligand_data=True --local_cpus=4`,
   plus any per-visit arguments from `<visit>/.user.yaml`:
   ```yaml
   autoprocessing:
     pandda:
       some_key: some_value    # becomes --some_key=some_value
   ```
   With `overwrite`, `processed_datasets/<dtag>` is removed first.
2. **Event selection** — reads `events.yaml`, takes the single highest-`Score`
   event. More than one binder per dataset is rare and the score ranks well.
3. **Map/model restriction before Rhofit** — the event map is masked to a
   radius around the event centroid (`mask_map`) and atoms within 10 Å of the
   centroid are stripped from the input model (`remove_nearby_atoms`). Both
   matter: Rhofit hunts non-binding-site density otherwise, and it masks the
   protein before building, so a model clipping the event makes autobuilding
   impossible.
4. **Rhofit** — `pandda_rhofit.sh` with a sigma-derived cut level, 1 h timeout.
5. **Scoring** — `ligand_score.py` per build against the z-map; PanDDA2's own
   internal autobuild is included in the comparison, scored with the event score.
   The winner is written to `best_score.txt` (which is what collate buckets on).
6. **Merge** — best ligand merged into the protein model as
   `modelled_structures/<dtag>-pandda-model.pdb`; any previous model is kept as
   `pandda-internal-fitted.pdb`; waters clashing with the ligand are removed.
7. **Output** — cropped event/z maps, an MVS html viewer, `pandda2_results.json`.

Failure at stages 1–2 returns `False` after uploading whatever attachments exist;
a Rhofit failure is logged but does **not** abort (PanDDA2's internal autobuild
may still carry the dataset).

## `pipedream_xchem` — `pipedream_xchem.py`

`job_parameters`: `dcid`, `processing_directory`, `dtag`, `n_datasets`,
`scaling_id`, `database_path`, `timeout-minutes`. Same array handling as above.

Inputs from `model_building/<dtag>`: `dimple.pdb`, `dimple.mtz`,
`<dtag>.free.mtz`, `compound/<code>.cif`. `process_pdb_file()` strips
crystallisation components from the dimple pdb first.

```
pipedream -nolmr -hklin <dtag>.free.mtz -xyzin dimple.pdb -hklref dimple.mtz
          -d <pipedream>/<dtag>
          -mrefine TLSbasic,WaterUpdatePkmaps -keepwater -remediate
          -sidechainrebuild -runpepflip -rhocommands -xclusters -nochirals
          -rhofit <code>.cif
```

BUSTER build is pinned in the wrapper (`/dls_sw/apps/GPhL/BUSTER/20250717`),
separate from the `buster/20260424` module the recipe loads.

Post-processing (each step degrades gracefully, returning `True` with whatever
was produced): `gemmi sf2map` for 2fofc/fofc maps → cropped map + MVS html →
`edstats` over the resolution range read back from `pipedream_summary.json`.

Outputs of interest: `postrefine-<code>/refine.pdb` / `refine.mtz`,
`report-<code>/report.pdf`, `summary.out`, `edstats.out`,
`pipedream_summary.json`.

> BUSTER's pipedream is Perl and its `-r` file tests read stat bits, **not**
> ACLs — a file readable via an inheritable ACL still gives "Couldn't find input
> pdb file". The fix is `chmod -R o+rX`, never `g+`.

## `xchem_collate` — `xchem_collate.py`

`job_parameters`: `processing_directory`, `pipedream`, `overwrite`,
`timeout-minutes`. Runs once per visit, not per dataset.

1. **PanDDA2 postrun** — `postrun.py --data_dirs=<model_building>
   --out_dir=<panddas> --use_ligand_data=False`, producing the events/sites csvs.
2. **soakDB re-integration**
   - `prepare_auto_db(processing_dir)` copies the master soakDB to
     `auto/database/autosoakDBDataFile.sqlite` if absent, then syncs any new
     columns and any new `mainTable` rows from master into the copy. **All
     writes go to the copy, never the master.**
   - `updatable_crystals(db_copy, overwrite)` returns the set of CrystalNames
     this run may write. It is the single gate — computed once, used both for
     building the update dicts and for the bulk write.
   - `symlink_score_buckets()` symlinks each updatable dataset into
     `pandda2/score_0.6-0.8/`, `score_0.8-0.9/`, `score_0.9-1/` by its
     `best_score.txt`, mirroring the panddas layout (`processed_datasets/` +
     a filtered `analyses/pandda_analyse_events.csv`) so models can be browsed
     by ligand score. Earlier batches' symlinks are preserved.
   - `update_xchem_database()` performs model selection and bulk-writes into
     the copy's `mainTable` (see below).
3. **Pipedream collate** (if `pipedream`) — `collate_pipedream_results.py` from
   the xchem micromamba env over `Pipedream_output.json`, then
   `write_pipedream_parameters()`.
4. **XChemAlign** — if `processed/analysis/xchemalign/{config,assemblies}.yaml`
   exist, copies them to `auto/xchemalign/` and runs
   `xchemalign.collator` then `xchemalign.aligner`.
5. **Cleanup** — `cleanup_setvar_files()` removes orphaned autoBUSTER setvar logs.

Every stage is individually wrapped in try/except: collate logs and continues
rather than failing the job, so a `success` status does not mean every stage ran.
Read the job log, not just the ISPyB status.

### Model selection (`update_xchem_database`)

Per dataset, in `model_building`, skipping anything not in `updatable` or with
no `compound/*.cif`:

| Condition | Selected | `RefinementOutcome` | `RefinementLigandConfidence` |
|---|---|---|---|
| `refine.pdb` exists and RSCC ≥ 0.8 | Pipedream | `4 - CompChem ready` | `4 - High Confidence` |
| `refine.pdb` exists and 0.7 < RSCC < 0.8 | Pipedream | `3 - In Refinement` | `2 - Correct ligand, weak density` |
| No Pipedream model above threshold, PanDDA2 model exists | PanDDA2 | `2 - PANDDA model` | from `pandda_ligand_confidence()` |
| Neither | — | row still written, no model | — |

The model is `pipedream/<dtag>/postrefine-*/refine.pdb`; the RSCC is the best
value in `rhofit-*/Hit_corr.log`, and selection needs it strictly above
`rscc_thresh = 0.7`. Only on the Pipedream branch are the full refinement /
validation statistics pulled from `pipedream_summary.json` and the files
exported into the dataset dir. All rows are stamped
`LastUpdated_by = "gda2"`, which is what `overwrite` mode keys on later.

> `RefinementLigandCC` is the **pre-refinement** omit-map Rhofit CC from
> `Hit_corr.log`, not the post-refinement summary ligandcc — deliberately, to
> avoid model bias.

Relevant helper modules: `dlstbx.util.soakdb`, `dlstbx.util.pandda`,
`dlstbx.util.xchem_collate_helpers`, `dlstbx.util.pipedream_xchem_helpers`,
`dlstbx.util.symlink`.