# 9 - Bring back into CAM

## Code modifications
Bring the CCPP version back into a new CAM clone or branch

- Make a new clone or branch of CAM6 (personal preference)
- Remove the non-CCPP version from your CAM source directory
- Make a new directory in the atmosperhic_physics submodule of CAM (make sure the directory name is the one you used in step one of the conversion to CCPP):
```
mkdir src/atmos_phys/schemes/<parameterization>
```
- Copy your new CCPP version files from CAM-SIMA into this new directory
- Update the routine names to match the CCPP-ized subroutine names where the subroutines are called
- Update the calling lists if they were changed when the CCPP-ized subroutines were made
- If the CCPP-ized module is in a new subdirectory, you will need to add it to `bld/configure`:
```
print $fh "$camsrcdir/src/atmos_phys/schemes/<parameterization>\n";
```

!!!Note "Handling readnl and diagnostics in CAM"
    The original `<parameterization>_readnl` subroutine (if there was one) must still be called in runtime_opts.F90 and all addfld/outfld calls must be preserved. If your new CCPP interfaces are called directly by physpkg, this may mean that you need to create a `<parameterization>_cam.F90` module that contains `<parameterization>_readnl`. 

## Testing

- To quickly test that your code is working during development, you can create two new model cases with a compset that involves your physics scheme, and the following XML changes:

```
./xmlchange --force ROF_NCPL=\$ATM_NCPL
./xmlchange DOUT_S=FALSE,STOP_OPTION=nsteps,STOP_N=9,DEBUG=TRUE
```

!!! Note - test cases
    The first case will be CAM without your changes, and the second will be CAM with your new physics scheme code changes. Both simulations should produce a `<case_name>.cam.rh0.XXX` file, which you can compare by using the cprnc tool, which can be found (and used) on derecho below. Your new physics code is working correctly if the printed output from cprnc states that there are no differences in the files.

```
/glade/campaign/cesm/cesmdata/cprnc/cprnc
cprnc <file1> <file2>
```
- Optional step: if want to have additional testing... Once you believe it is working, make a [snapshot](create-snapshots.md) to run CAM with this modified clone.
    - Use the cprnc tool to compare the resulting CAM6 “after” snapshot file with your original CAM6 “after” snapshot file (from before the port). If they differ and you changed the internal calculations during your port, you will need to determine if these changes are expected and correct.  
        - If they are different (and the changes are correct), then you will need to repeat the steps listed in the [Run CAM-SIMA](run-cam-sima.md) section with the newly generated snapshot files from the modified CAM clone.

Proceed to [Step 10 - Final Steps](final-steps.md)