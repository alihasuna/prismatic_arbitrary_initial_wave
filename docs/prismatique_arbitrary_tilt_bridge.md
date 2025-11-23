# Passing arbitrary HRTEM tilts through prismatique → pyprismatic → prismatic

This note links the three layers involved in an HRTEM tilt request so you can wire prismatique's `Params` objects to the C++ S-matrix code that performs the sampling.

## Data flow summary
- **prismatique** surfaces tilt settings via `prismatique.tilt.Params` and ultimately calls **pyprismatic** to launch a simulation.
- **pyprismatic** converts its `Metadata` fields to C++ via `pyprismatic.core.go`, scaling milliradian inputs to radians and copying them into `Prismatic::Metadata` (`probeXtilt`, `probeYtilt`, rectangular/radial ranges, and the `xTiltStep`/`yTiltStep` spacing that previously enforced interpolation grid alignment).【F:pyprismatic/core.cpp†L27-L226】
- **prismatic (C++)** consumes those metadata fields in `PRISM02_calcSMatrix.cpp`, where tilt selection and S-matrix assembly happen. Any change that allows arbitrary (non-grid-aligned) tilts should therefore read the same metadata and drop the modulus checks that currently snap to the interpolation grid.

## How to plumb arbitrary tilts end-to-end
1. **Accept arbitrary angles in prismatique**: ensure the prismatique parameter object sends milliradian values for `probeXtilt`/`probeYtilt` (single-probe tilts) and the HRTEM selection ranges. No snapping should be performed in Python; prismatique only needs to serialize these fields to the `pyprismatic` input JSON/args.
2. **Keep the same field names in pyprismatic**: the Python wrapper already exposes the tilt fields (`probeXtilt`, `probeYtilt`, `minXtilt`, `maxXtilt`, `minYtilt`, `maxYtilt`, `xTiltStep`, `yTiltStep`, offsets) on the `Metadata` object and lists them in the fields that are transmitted to C++.【F:pyprismatic/params.py†L120-L168】【F:pyprismatic/params.py†L294-L305】 No schema change is required—just allow non-grid values to pass through untouched.
3. **Update the C++ selector/S-matrix builder**: modify `PRISM02_calcSMatrix.cpp` so `setupBeams_HRTEM` interprets the incoming tilts as free-form values (e.g., fractional indices with interpolation weights) instead of enforcing `interp_fx`/`interp_fy` divisibility. Once that is done, the pyprismatic arguments above will naturally drive arbitrary tilts from prismatique down to the S-matrix.
4. **Export the selected tilts to HDF5**: the existing HDF5 writer in `src/fileIO.cpp` persists `xTilts_tem`/`yTilts_tem` to the `dim3`/`dim4` datasets. Confirm these arrays reflect the new arbitrary angles so prismatique can read them back for visualization or downstream processing.

## Quick wiring checklist
- Use prismatique's `Params` to set milliradian tilt values; avoid rounding to interpolation factors.
- Ensure pyprismatic receives those values unchanged (e.g., by logging `Metadata.probeXtilt`/`probeYtilt` before calling `pyprismatic.core.go`).
- After updating the S-matrix code, verify the written `dim3`/`dim4` datasets match the requested angles to confirm the full stack honors arbitrary tilts.

## Is arbitrary tilting “done” yet?
Not yet. The Python layers already pass through arbitrary angles, but the C++ S-matrix selection still needs the interpolation-agnostic changes described above. Until `setupBeams_HRTEM` (in `src/PRISM02_calcSMatrix.cpp`) is updated, tilts will remain snapped to the interpolation grid.

## How to test once the C++ changes land
1. Choose a tilt that is **not** an integer multiple of `1/interp_fx` or `1/interp_fy` (e.g., `probeXtilt=3.7 mrad`, `probeYtilt=5.1 mrad`).
2. Run pyprismatic with that tilt from prismatique or directly via `pyprismatic.core.go`, and enable HDF5 output (`--writeHDF` or equivalent) to capture the S-matrix metadata.
3. Inspect the resulting HDF5 file (datasets `dim3` and `dim4`) and confirm the stored tilt arrays include the requested angles rather than the nearest interpolation-aligned values.
4. For an image-level sanity check, render two simulations at nearby fractional tilts (e.g., `3.7/5.1 mrad` vs. `3.9/5.3 mrad`) and verify the outputs differ smoothly instead of in grid steps, indicating the S-matrix interpolation is working.
