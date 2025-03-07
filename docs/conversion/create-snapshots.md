# 6 - Create snapshots of CAM

## CAM snapshot run tips

1. Try to use the lowest resolution grid possible in order to keep the snapshot file size small (e.g. `ne3pg3_ne3pg3_mg37`).
1. Run with debug flags on (`./xmlchange DEBUG=True`), as this way errors that can occur when you bring the CCPP-ized code back into CAM are easier to catch. Make sure ALL CAM and CAM-SIMA runs have debug on.
1. You should generate at least one set of snapshot files using GNU on derecho with DEBUG=TRUE, as these will be saved for use in the CAM-SIMA physics testbed to prevent unexpected answer changes.

## Configure and set-up CAM snapshot
Make a normal **CAM** (not CAM-SIMA) run using cam_snapshot to capture before and after files using a compset which exercises the parameterization being converted.

- The cam_snapshot feature dumps the entire state/tend/pbuf/cnst arrays to netCDF files.
    - The variables `cam_snapshot_before_num` and `cam_snapshot_after_num` are the output file numbers (offset by 1 as they are in CAM).
    - The `cam_take_snapshot_before` and `cam_take_snapshot_after` are usually set to the same parameterization to capture the variables before and after the named parameterization.
        - To find the snapshot name, look in your source code checkout in `bld/namelist_files/namelist_definition.xml`.
            - Inside this file, find the “cam_take_snapshot_before” definition.  In this definition, it lists all possible parameterizations which can have a snapshot file created around.
                - If your parameterization is there, that will be your `cam_take_snapshot_before` and `_after` values in your `user_nl_cam`
                - If it's not there, you'll add `user_set` snapshot calls to `physpkg.F90` per the instructions [here](#user-set-snapshot-calls)



- To enable cam_snapshot, add the following to your user_nl_cam file:
```
cam_snapshot_before_num=6
cam_snapshot_after_num=7
cam_take_snapshot_before='<parameterization>_tend' ! or 'user_set'
cam_take_snapshot_after='<parameterization>_tend' ! or 'user_set'
nhtfrq = 0,0,0,0,0,1,1
ndens = 2,2,2,2,2,1,1
```

- The “before” file will be used for input for testing; “after” file will be used for comparisons
- Make sure to avoid using tape number one (h0) for `cam_snapshot_before_num` and `cam_snapshot_after_num`
- Check your namelist for any extra “fincl” settings added by use cases, as those will add variables that can conflict with the snapshot output.
    - To check this, before you modify the `user_nl_cam` file,  run `./case.setup`, run `./preview_namelists` and then look at the resulting `atm_in` file in the run directory. Look for `finclX` where X is your possible “before” and “after” numbers.  Choose a number that is not used.
        - You most likely will see fincl1 which is why it is not an option. 
- Set nhtfrq to 1 so that data is written out on every time step.  Note that the nhtfrq and ndens variables are arrays based off of the “tape” number specified in `cam_snapshot_before` and `cam_snapshot_after`.  (i.e. 6 = the sixth number in these two arrays)
- Finally, as cam_snapshot writes out all of state, pbuf and other variables at every time step, it is suggested that a limited number of time steps be made in a cam_snapshot run.
    - The run must be at least 2 time steps in order to work in subsequent steps. We recommend 3 to at most 9 time steps.  (Note a 3 timestep run using a 2 degree FV grid can produce snapshot files with sizes over 4 Gb each).
    - To change the number of timesteps to 3 use `./xmlchange STOP_OPTION=nsteps` and `./xmlchange STOP_N=3`

!!! Note "Coupling frequency"
    With short NUOPC runs, you may need to change ROF_NCPL and/or GLC_NCPL to the same size at ATM_NCPL to get them to run properly (`./xmlchange ROF_NCPL=\$ATM_NCPL` and `./xmlchange GLC_NCPL=\$ATM_NCPL`)

## Create CAM snapshots
Run CAM normally with 
```
./case.setup
./case.build
./case.submit
```
Upon a successful completion, you will see a `cam.h5i.xxx.nc` and `cam.h6i.xxx.nc` in your run directory (assuming you used before and after of 6 and 7)

## Save snapshot files
The generated snapshot files should be saved on **derecho** here:
```
/glade/campaign/cesm/community/amwg/sima_baselines/cam_sima_test_snapshots
```

with the naming convention `cam_<resolution>_<parameterization>_snapshot_derecho_<compiler>_before_cYYYYMMDD.nc` and `cam_<resolution>_<parameterization>_snapshot_derecho_<compiler>_after_cYYYYMMDD.nc`

Be sure to run
```
chmod u=rw,g=r,o=r <snapshot_files>
```
or
```
chmod 644 <snapshot_files>
```
on every file you copy over so that it is readable by everyone

On **izumi**, save your converted snapshot files in:
```
/project/amp02/cam_snapshot_files/<parameterization>
```

Once you have created your snapshot files, proceed to [7 - Run CAM-SIMA](run-cam-sima.md)

## "user set" snapshot calls
Depending on where your scheme is called within CAM physics (in CAM6 or CAM7 physpkg and in tphysac or tphysbc), choose a code snippet below.

!!! Warning "Snapshot call changes"
    If the snapshot calls change in CAM, this section may not be updated immediately. If you're running into issues, mirror an existing set of snapshot calls in the physpkg.F90 routine you are working in.

### CAM7 tphysbc

In `$CAM/src/physics/cam7/physpkg.F90`:

```
if (trim(cam_take_snapshot_before) == "user_set") then
   call cam_snapshot_all_outfld_tphysbc(cam_snapshot_before_num, state, tend, cam_in, cam_out, pbuf, &
           cmfmc, cmfcme, zdu, rliq, rice, dlf, dlf2, rliq2, net_flx)
end if

<call to parameterization run phase>

if ( (trim(cam_take_snapshot_after) == "user_set") .and. &
     (trim(cam_take_snapshot_before) == trim(cam_take_snapshot_after))) then
        call cam_snapshot_ptend_outfld(ptend, lchnk)
end if

<call to physics_update and/or ptend outfld calls>

if (trim(cam_take_snapshot_after) == "user_set") then
   call cam_snapshot_all_outfld_tphysbc(cam_snapshot_after_num, state, tend, cam_in, cam_out, pbuf, &
           cmfmc, cmfcme, zdu, rliq, rice, dlf, dlf2, rliq2, net_flx)
end if
```

### CAM7 tphysac
In `$CAM/src/physics/cam7/physpkg.F90`

```
if (trim(cam_take_snapshot_before) == "user_set") then
   call cam_snapshot_all_outfld_tphysac(cam_snapshot_before_num, state, tend, cam_in, cam_out, pbuf,&
                    fh2o, surfric, obklen, flx_heat, cmfmc, dlf, det_s, det_ice, net_flx)
end if

<call to parameterization run phase>

if ( (trim(cam_take_snapshot_after) == "user_set") .and.      &
     (trim(cam_take_snapshot_before) == trim(cam_take_snapshot_after))) then
   call cam_snapshot_ptend_outfld(ptend, lchnk)
end if

<call to physics_update and/or ptend outfld calls>

if (trim(cam_take_snapshot_after) == "user_set") then
   call cam_snapshot_all_outfld_tphysac(cam_snapshot_after_num, state, tend, cam_in, cam_out, pbuf,&
                    fh2o, surfric, obklen, flx_heat, cmfmc, dlf, det_s, det_ice, net_flx)
end if
```

### CAM6 tphysbc
In `$CAM/src/physics/cam/physpkg.F90`:
```
if (trim(cam_take_snapshot_before) == "user_set") then
   call cam_snapshot_all_outfld_tphysbc(cam_snapshot_before_num, state, tend, cam_in, cam_out, pbuf, &
           flx_heat, cmfmc, cmfcme, zdu, rliq, rice, dlf, dlf2, rliq2, det_s, det_ice, net_flx)
end if

<call to parameterization run phase>

if ( (trim(cam_take_snapshot_after) == "user_set") .and.      &
     (trim(cam_take_snapshot_before) == trim(cam_take_snapshot_after))) then
   call cam_snapshot_ptend_outfld(ptend, lchnk)
end if

<call to physics update and/or ptend outfld calls>

if (trim(cam_take_snapshot_after) == "user_set") then
   call cam_snapshot_all_outfld_tphysbc(cam_snapshot_after_num, state, tend, cam_in, cam_out, pbuf, &
           flx_heat, cmfmc, cmfcme, zdu, rliq, rice, dlf, dlf2, rliq2, det_s, det_ice, net_flx)
end if
```

### CAM6 tphysac
In `$CAM/src/physics/cam/physpkg.F90`: 

```
if (trim(cam_take_snapshot_before) == "user_set") then
   call cam_snapshot_all_outfld_tphysac(cam_snapshot_before_num, state, tend, cam_in, cam_out, pbuf,&
                    fh2o, surfric, obklen, flx_heat)
end if

<call to parameterization run phase>

if ( (trim(cam_take_snapshot_after) == "user_set") .and.      &
     (trim(cam_take_snapshot_before) == trim(cam_take_snapshot_after))) then
   call cam_snapshot_ptend_outfld(ptend, lchnk)
end if

<call to physics update and/or ptend outfld calls>

if (trim(cam_take_snapshot_after) == "user_set") then
   call cam_snapshot_all_outfld_tphysac(cam_snapshot_after_num, state, tend, cam_in, cam_out, pbuf,&
                    fh2o, surfric, obklen, flx_heat)
end if
```
