Openbind Diff2IR
===================================

**Diff2IR** is the Openbind automated pipeline project for automatically transforming diffraction data upon collection to complete bound state models in Fragalysis.

.. note::

   This project is under active development.

:doc:`workflow` is the place to start: it walks through the three stages of the
pipeline, from beamline data collection to Fragalysis upload. The remaining pages
document individual components in more detail, and :doc:`dev/index` covers how the
automation is wired together for those working on the pipeline itself.

Contents
--------

.. toctree::
   :maxdepth: 2

   workflow
   dev/index
   githubs

.. toctree::
   :maxdepth: 1
   :caption: Components

   autopandda
   data_distribution
   high_occupancy_model_completion
   ligand_autobuilding
   autobuild_scoring
   curation
   low_occupancy_model_completion
   water_completion
   model_selection
