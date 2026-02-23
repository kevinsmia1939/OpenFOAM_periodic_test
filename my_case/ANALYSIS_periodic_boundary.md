# Periodic boundary troubleshooting (`my_case` vs `periodicHill`)

## What failed and why

Your latest logs show two separate issues:

1. `meshCase/system/createPatchDict` was malformed/truncated (missing `FoamFile` header), so `createPatch` aborted immediately with a header read error.
2. In `case`, `cyclicAMI` still failed during parallel `potentialFoam` with zero AMI weights and invalid target bounding box, even after fixing translation distance.

## Why the AMI failure can still happen

`checkMesh` in serial can report healthy AMI weights, but `potentialFoam -parallel` may still fail if AMI matching is fragile in decomposition/runtime for this specific mesh and patch distribution.

For your geometry, the two periodic side patches are planar and corresponding, i.e. this is a conformal periodic pair, so `cyclic` is a better fit than `cyclicAMI`.

## Implemented fix

To match the `periodicHill` strategy and remove AMI sensitivity, the periodic pair has been switched from `cyclicAMI` to `cyclic`:

- `my_case/case/system/createPatchDict`
- `my_case/case/0/U`
- `my_case/case/0/p`

Additionally, `my_case/meshCase/system/createPatchDict` has been fully restored with a valid OpenFOAM dictionary header and aligned periodic settings.

## Notes on warnings

- `Cannot find any patch ... patch_1_.*`/`patch_2_.*` means patch regex does not match current boundary names at that stage.
- If that warning appears again, inspect `constant/polyMesh/boundary` before `createPatch` and update regex to exact names.

## Suggested rerun order

From `case`:

1. `createPatch -overwrite`
2. `checkMesh -allTopology -allGeometry`
3. run `./Allrun`

If it still fails, share:

- `system/createPatchDict`
- `0/U`, `0/p`
- `constant/polyMesh/boundary` right after `createPatch -overwrite`
- the exact `potentialFoam` command line from `Allrun`
