# OpenWQ functional unit tests (synthetic test cases)

Configuration files and scripts for the functional unit tests of OpenWQ coupled to SUMMA and mizuRoute,
as reported in Costa et al. (submitted to the *Journal of Advances in Modeling Earth Systems*,
Appendix C, Tables C1–C2 and Figs. 5–7, C10–C12). Each test compares OpenWQ against an analytical
solution: batch reaction networks (Tests 1–6) and advection–dispersion with and without first-order
decay in a SUMMA soil column and a mizuRoute river network (Tests 7–10).

## Test cases

| Paper test | Folder | Process | Hosts |
|---|---|---|---|
| 1 | `9_batch_singleSp_1storder` | single species, first-order decay | SUMMA, mizuRoute |
| 2 | `10_batch_singleSp_2ndorder` | single species, second-order decay | SUMMA, mizuRoute |
| 3 | `11_batch_2species` | two species in series | SUMMA, mizuRoute |
| 4 | `11_1_batch_3species` | three species in series | SUMMA, mizuRoute |
| 5 | `12_batch_nitrogencycle` | nitrogen cycle (NREF, NLAB, DON, DIN) | SUMMA, mizuRoute |
| 6 | `13_batch_oxygenBODcycle` | DO–BOD (Streeter–Phelps) | SUMMA, mizuRoute |
| 7 | `4_nrTrans_contS_PorMedia` | continuous source, non-reactive transport | SUMMA, mizuRoute |
| 8 | `8_nrTrans_contS_PorMedia_linDecay` | continuous source, transport + first-order decay | SUMMA, mizuRoute |
| 9 | `2_nrTrans_instS_PorMedia` | instantaneous source, non-reactive transport | SUMMA |
| 10 | `6_nrTrans_instS_PorMedia_linDecay` | instantaneous source, transport + first-order decay | SUMMA |

Each test folder holds a `summa/` and/or `mizuroute/` run directory with the OpenWQ master file
(`openWQ_master.json`), the OpenWQ configuration and module files, and the host-model inputs.
The `crhm/` folders hold the original CRHM–OpenWQ configurations; they have not been migrated to the
current OpenWQ input format.

`99_analytical_solutions/` contains the MATLAB scripts that compute the analytical solutions
(`Reactions_SyntheticTests.m` for the batch tests, `AnltSOL_TRANSIENT_summa.m` and
`AnltSOL_TRANSIENT_mizuroute.m` for transport). The root-level MATLAB scripts read OpenWQ, SUMMA and CRHM
outputs. `99_support/diagrams/` holds the reaction-network diagrams.

## Required model versions

| Component | Version |
|---|---|
| OpenWQ | v1.1.1 or later (`ue-hydro/openwq`, branch `master`), doi:10.5281/zenodo.23183928 |
| SUMMA | `ue-hydro/summa_ashleymedin`, branch `develop` |
| mizuRoute | `ue-hydro/mizuRoute_ESCOMP_withOpenWQlink`, branch `main`, commit `a181364` or later |

OpenWQ v1.1.0 and earlier releases of 2026 do not reproduce Tests 7–10 (advected-fraction formulation in
dissolved transport), and mizuRoute `main` before `a181364` does not inject the runoff solute into the
river network.

## Building (Docker)

Start the OpenWQ container (`openwq/containers`, `docker compose up -d`) and, inside it, build from the
OpenWQ folder placed in the host model:

```bash
# SUMMA-OpenWQ (OpenWQ cloned into summa/build/source/openwq/openwq)
cmake -DHOST_MODEL_TARGET=summa_openwq -DCMAKE_BUILD_TYPE=Release . && make -j8
# mizuRoute-OpenWQ (OpenWQ cloned into route/build/openwq/openwq)
cmake -DHOST_MODEL_TARGET=mizuroute_lakes_openwq -DCMAKE_BUILD_TYPE=Release . && make -j8
```

## Running a test

From the test's host folder (OpenWQ reads `openWQ_master.json` from the working directory):

```bash
# SUMMA
cd 9_batch_singleSp_1storder/summa
mkdir -p summa/output
summa_openwq_Release -m summa/SUMMA/summa_fileManager_OpenWQ_systheticTests_BGQ.txt

# mizuRoute (at least 2 MPI processes)
cd 9_batch_singleSp_1storder/mizuroute
mkdir -p mizuroute_out
mpirun -np 2 mizuroute_lakes_openwq_Release mizuroute_in/settings/openwq_syntheticTest.control
```

Results are written to `Output_OpenWQ/HDF5/`. In mizuRoute output the reach order follows the MPI
decomposition; select reaches by the `reachID` dataset, not by column position.

## Notes on the setup

- Solver: `FORWARD_EULER`, as in the paper. mizuRoute runs with a daily step, so its batch results are
  less accurate than SUMMA's (for example Test 6 DO, NSE 0.976); `"SOLVER": "SUNDIALS"` gives NSE 0.999.
- mizuRoute Tests 7–8 use `<hw_drain_point> 1` (headwater runoff enters at the top of the reach) and a
  zero initial concentration, matching the analytical problem c(x, 0) = 0. With the mizuRoute default
  (`2`), OpenWQ is not called for headwater reaches and the single runoff source never enters the network.
- The analytical transport solutions use depth in mm, with D = 1e-4 mm2 s-1 (effectively advection only).
- `parfor_progressbar.m` is a third-party MATLAB File Exchange contribution, used only to display progress
  in the SUMMA output readers.

## Expected results (NSE against the analytical solutions)

| Test | SUMMA | mizuRoute |
|---|---|---|
| 1–5 (all species) | 1.0000 | 0.999–1.000 |
| 6 (DO, BOD) | 1.0000 | 0.976, 0.998 |
| 7 (120 d / reaches at 80, 194, 328 km) | 0.9875 | 0.978, 0.981, 0.983 |
| 8 (120 d / reaches at 80, 194, 328 km) | 0.9955 | 0.974, 0.975, 0.925 |
| 9 (120 d) | 0.9536 | – |
| 10 (120 d) | 0.9523 | – |

Analytical solutions follow Wexler (1992, USGS TWRI 3-B7) and van Genuchten and Alves (1982).

## License

Creative Commons Attribution 4.0 International (CC BY 4.0), see `LICENSE`.
