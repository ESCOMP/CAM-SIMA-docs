# 7 - Check metadata

- Create and configure a short test case in CAM-SIMA that exercises your new scheme. You may find it easiest to use the FPHYStest compset (using the null dycore). For more on setting up a case, see [Create a case](../usage/creating-a-case.md)
    
```
./create_newcase --case <casedir> --compset FPHYStest --res ne3pg3_ne3pg3_mg37 --compiler <compiler> --run-unsupported
cd <casedir>
./xmlchange CAM_CONFIG_OPTS="--dyn none --physics-suites <parameterization>"
./xmlchange STOP_OPTION="nsteps"
./xmlchange STOP_N=<some # of steps less than your snapshot file contains>
./xmlchange DOUT_S=False
```

- Run `./preview_namelists` to see if the framework will generate the caps. A few possible outcomes:
    - There are no errors and you are somehow a perfect person. Move on to the [next step](run-cam-sima.md) and ultimately run the model
    - You get an error
        - Determine what the course of action should be based on the table below. Once you think you have addressed the issue, rerun `./preview_namelists` and repeat!

<table markdown>
<thead markdown>
<tr markdown>
   <th markdown>Error</th>
   <th markdown>Situation</th>
   <th markdown>Action</th>
</tr>
</thead>
<tbody markdown>
<tr markdown>
   <td markdown style="vertical-align:middle" rowspan="5">`Input argument for <scheme>_<phase>, <standard_name>, not found`</td>
   <td markdown>Standard name incorrect when compared against the spreadsheet</td>
   <td markdown>Correct standard name</td>
</tr>
<tr markdown>
   <td markdown>Variable traced back to "use" statement in CAM</td>
   <td markdown>Follow [procedure 1](#procedure-1) below</td>
</tr>
<tr markdown>
   <td markdown>Variable is a constituent (can be traced back to the `state%q` array in CAM)</td>
   <td markdown>Add `advected = true` to the variable's metadata in your parameterization</td>
</tr>
<tr markdown>
   <td markdown>Variable can be traced back to a calculation in the CAM interface code</td>
   <td markdown>Make an [interstitial](interstitials.md)</td>
</tr>
<tr markdown>
   <td markdown>None of the above criteria are met</td>
   <td markdown>Follow [procedure 2](#procedure-2) to add the variable to the registry</td>
</tr>
<tr markdown>
   <td markdown>`parse_source.ParseSyntaxError: Unsupported unit conversion, '<unit1>' to '<unit2>' for '<standard_name>'`</td>
   <td markdown>Units do not match and there is not automatic unit converter for the conversion</td>
   <td markdown>Follow [procedure 3](#procedure-3)</td>
</tr>
<tr markdown>
   <td markdown>`parse_source.CCPPError: Invalid 'standard_name' property value`</td>
   <td markdown>Standard name is not parsing correctly</td>
   <td markdown>Look for a missing "_" or a typo in the standard name</td>
</tr>
<tr markdown>
   <td markdown>`CCPPError: Could not find dimension, <dimension1>:<dimension2>, in <variable>`</td>
   <td markdown>Dimension is either invalid or not implemented in CAM</td>
   <td markdown>Confirm that the dimension is valid and that you selected the correct one (e.g. `vertical_layer_dimension` vs `vertical_interface_dimension`. If it's valid and still missing, consult with the other CAM SEs</td>
</tr>
<tr markdown>
   <td markdown>`parse_source.ParseInternalError: Duplicate Group variable <variable>`</td>
   <td markdown>You have multiple variables (with different standard names) that have the same local name</td>
   <td markdown>Change the local name of one (or more) of the variables in your Fortran and metadata</td>
</tr>
<tr markdown>
    <td markdown>`Unknown DDT type`</td>
    <td markdown>The framework does not know about a derived data type.</td>
    <td markdown>Create a metadata file for the DDT (if it does not already exist). Add the path to the metadata file that defines the DDT to the top of the registry</td>
<tr markdown>
   <td markdown>Some other misc error</td>
   <td markdown>You've run into either a bug or just a weird scenario</td>
   <td markdown>If the error message is unhelpful, ask Courtney</td>
</tr>
</tbody>
</table>

Once you have validated your metadata, proceed to [8 - Run CAM-SIMA](run-cam-sima.md)

## Procedure 1
If the CAM variable can be traced back to a use statement, you may have to create or amend a SIMA-side metadata file.

- Try to find the equivalent module and the "used" variable within it in CAM-SIMA
    - If you find it the module:
        - It already has a metadata file, move the variable in the Fortran to be with the other variables that have metadata. Then, add your variable to the appropriate place in the metadata file.
        - It doesn't have a metadata file, create one and add your variable. You'll also have to add the ["html" tags](convert-portable-layer.md#1b-add-required-htmlinclude-lines) and potentially rearrange module-level variables so the only variable(s) under the tags are the ones you're providing metadata for.
           - Then add the path to the metadata file to the `metadata_file` section of `$CAM-SIMA/src/data/registry.xml`
    - If you can't find the module, consult with the other CAM SEs

## Procedure 2
If the variable in question cannot be tracked back to a “use” statement or a calculated value in the CAM interface code, it likely needs to be provided by the host model. To enable this, you’ll have to add the variable to the registry (`$CAM-SIMA/src/data/registry.xml`). Here is the structure of how that will look:

```
<variable local_name="phis"         
              standard_name="surface_geopotential"
              units="m2 s-2" type="real" kind="kind_phys"
              allocatable="allocatable">
      <dimensions>horizontal_dimension</dimensions>
      <ic_file_input_names>phis state_phis</ic_file_input_names>
</variable>

```

- If the variable you are adding is a state or ptend variable, be sure to also add the variable standard name to the relevant ddt in the registry.
- `dimensions` is a space-separated list (in order) of the standard names of the dimensions for the variable
- `ic_file_input_names` is a space-separated list of possible names on the snapshot/inputdata file

!!! Note - registry variables
    Variables in the registry are read in from the initial data file by default.

!!! note "Should I add a variable to the registry or rely on CCPP to manage the variables being passed between schemes?"
    While declaring a variable in the scheme metadata files allows CCPP to pass this variable from a scheme and to another scheme without having to define it in the registry, it is important to note that variables not declared in the registry are not preserved between timesteps (i.e., the timestep initialization phase will reallocate these variables). Thus variables that should be preserved as part of model "state" or (formerly) physics buffer variables with time dimensions (e.g., total energy at end of previous physics timestep) should be included as part of the registry.

There are two options for registry variables:

1. Variable read in from the initial data file: no action required
2. Variable not read in from the initial data file, and instead initialized during CAM-SIMA initialization. You will need to add code to do this:
    - Determine the best place to initialize this variable (has to occur at some point during initialization - see [CAM-SIMA run](../design/cam-run-process.md/#cam_init)
    - In the location you have chosen, add the use statements below to import the registry variable from the auto-generated fortran AND the mark_as_initialized routine
    - After adding the code to initialize the variable, add the `mark_as_initialized` call below to indicate that that variable should NOT be read in from the file.

```
use physics_types, only <var_local_name>
use phys_vars_init_check, only: mark_as_initialized
```

```
<var_local_name> = <something>
call mark_as_initialized('<var_standard_name>')
```

## Procedure 3
The units you are using for a variable in your converted scheme has a different unit than the host model (or possibly different from elsewhere in your suite). 

- Do a grep in $CAM-SIMA/src for the standard name to see what the discrepancy is.
- Double-check that the units for your scheme match the units in the spreadsheet
- If it seems like the automatic unit converter should be able to handle the unit conversion, talk to Courtney
- If it’s more of a one-off conversion, two options remain:
    1. Write an interstitial for the unit conversion
    2. Include the conversion in the scheme code (you will also need to update the CAM interface to make sure it’s passing in the same units as the CAM-SIMA host has)
