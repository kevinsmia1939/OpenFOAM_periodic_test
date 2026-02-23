# Periodic boundary failure analysis (`my_case` vs `periodicHill`)

## Root cause found

The crash is caused by an incorrect **periodic AMI translation distance**:

- In `my_case`, both periodic patches are configured with
  `separationVector ( 50.0 0.0 0.0 );`.
- Your mesh coordinates are in meters around `x = ±0.025` (domain length about `0.05 m`), so the correct translation should be `0.05`, **not 50**.

With `50`, AMI attempts to match patches 50 m apart, gets zero overlap (`sum(weights)=0`), then `potentialFoam` later hits a floating-point exception.

## Evidence from your log

- `AMI: Patch source sum(weights) min:0 max:0 average:0`
- `AMI: Patch target sum(weights) min:0 max:0 average:0`
- Bounding-box mismatch warning where target box is invalid/empty.

These are classic symptoms of wrong `separationVector` (or wrong patch pairing).

## Comparison to `periodicHill`

- `periodicHill` uses **conformal** periodic boundaries (`cyclic`) in `blockMeshDict`; it does not rely on AMI overlap reconstruction for streamwise periodicity.
- Your case uses `cyclicAMI`, which is acceptable for non-conformal interfaces, but then the geometric transform must be exact.

## Extra warning you can ignore/fix

`createPatch` warns:

- `Cannot find any patch or group names matching patch_0_0`

This comes from the `defaultFaces` block trying to collect a patch that does not exist. It is not the direct crash cause, but it should be cleaned up to avoid confusion.

## Changes applied here

- Updated periodic translation in the runtime case template:
  - `my_case/case/system/createPatchDict`
- Replaced `defaultFaces` source selection from `("patch_0_0")` to empty `()` to remove the warning.

## Suggested next run

From your case directory:

1. Re-run patch creation and check boundary entries:
   - `createPatch -overwrite`
   - verify `constant/polyMesh/boundary` has `separationVector (0.05 0 0)`
2. Run mesh/boundary quality checks:
   - `checkMesh -allTopology -allGeometry`
3. Run your solver workflow again (`Allrun`).

If AMI still reports zero weights, then patch geometry itself is not corresponding (wrong patch IDs selected by regex), and the next step is to inspect the raw patch names in `constant/polyMesh/boundary` *before* `createPatch` and map master/slave from those explicit names.
