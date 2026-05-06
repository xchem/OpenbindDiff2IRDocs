Model Selection
===================================

The Diff2IR pipelines processes all datasets with all available pipelines, and then selects final structural models. The logic is:

1. If no ligand bound model has been built by any pipeline, then no structural model is produced for the dataset
2. If only one pipeline has produced a ligand bound model, then that result is used
3. If Pipedream has built any ligand with an RSCC > 0.7 against the DIMPLE processed mtz, then the final Pipedream structural model for the ligand with the highest RSCC is used 
4. Otherwise the PanDDA 2 structural model is used