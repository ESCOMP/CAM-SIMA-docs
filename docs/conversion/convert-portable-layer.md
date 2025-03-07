---
title: 1 - Convert the "portable" layer
---

# 1 - Convert the "portable" layer

## Move the portable layer to atmospheric_physics

The *portable* parameterization layer is the module that contains the core physics code that is called by the CAM interface.

The portable module will most likely live in `$CAM/src/physics/cam` or `$CAM/src/physics/cam7`

The driver code in each physics directory is `physpkg.F90`, which typically calls the CAM interface to the portable layer (often called `<parameterization>_cam.F90`), but also sometimes calls portable layer directly.
Once you locate the core code that you will be converting, copy it into the new directory you created in the atmospheric_physics submodule directory:
```
cp $CAM/src/physics/<physics_subdir>/<file>.F90 $CAM-SIMA/src/physics/ncar_ccpp/schemes/<parameterization_name>
```

## Optional: pre-split the module
Many CAM schemes have more than one "run" or "tend" method contained within them.  To split them into separate files and test them, do the following:

- In the atmospheric physics directory (`ncar_ccpp/schemes/<parameterization_name>`), create a separate module for each piece which has a "run" or "tend" method.
    - An easy way to see what routines need to be separated out, is to look at the "use" statement(s) in `physpkg.F90` for your parametrization's module.  If more than one routine is listed, you most likely will need to separate these out.
- If there is shared, module-level data or shared subroutines which are called internally, put these all in a <scheme_name>_common.F90 module.

## 1a - Change routine names
Convert the original routines (except readnl - we'll get to that [later](create-namelist-xml.md)!) in the file(s) you copied over to $CAM-SIMA/src/physics/ncar_ccpp to one or more of the following 5 subroutines:
!!! note "`<parameterization>` should be the full name of your module"

    For example, if you are converting the `tj2016` `precip_tend` function, then `<parameterization>` would be `tj2016_precip_tend`.

- **<parameterization\>_register**
    - Add all code that is run only during the first timestep (nstep=0) BEFORE the physics grid has been set up.
    !!! Warning "Grid is not set up!"
        Only scalar variables can be passed in at this time, as the grid is not set up (no horizontal or vertical dimensions are available)
    - If your scheme has runtime constituents, this is place to add those (see [Runtime Constituent Usage](../design/constituents.md/#run-time-dynamic-constituents))
- **<parameterization\>_init**
    - Add all code that is run only during the first timestep (nstep=0) AFTER the physics grid has been set up.
- **<parameterization\>_timestep_init**
    - Add all pre-processing code needed by the scheme at the start of each timestep. This may contain code from the CAM interface (`physpkg.F90`) routine which prepares data for the run routine at each timestep
- **<parameterization\>_run**
    - This is the workhorse routine which is run at each timestep. The bulk of your ported code will likely be here.
    - Often, the workhorse routine in CAM is called `<parameterization>_tend`
- **<parameterization\>_timestep_final**
    - Add all post-processing code needed by the scheme at the end of each timestep. This may contain code from the CAM interface routine (in `physpkg.F90`) which manipulates data after the run routine at each timestep
- **<parameterization\>_final**
    - Most current CAM routines do not have code in this category. This code is run once at the very end of the CAM model execution.

!!! note

    You may not need all of the routines listed, nor do you need to supply them if they are not needed; however, all subroutine input/output (dummy) arguments need to have an "intent" label.


## 1b - Add required `\htmlinclude` lines

Add two lines before each of the up to 5 phases of subroutines (skip `readnl`)
```
!> \section arg_table_<parameterization>_<phase> Argument Table
!! \htmlinclude <parameterization>_<phase>.html
```

## 1c - Clean up dummy argument dimensions
Make sure no input/output variables have named dimensions in their declaration inside the routines.  

1. Replace named dimensions with “:”
    - for example: `real(r8), intent(in) :: zm(pcols, pver)` will become `real(r8), intent(in) :: zm(:,:)`
1. If a named dimension is no longer used, it may be removed from the calling list (i.e. pcols, etc)
1. On the CAM interface side, any variables dimensioned by `pcols` in CAM will need to be subsetted to `1:ncol` in the call to the CCPP-ized routine so that only the active columns are passed
1. All `intent(out)` variables which are dimensioned pcols` should be initialized to zero right before being called in the main CAM physics calling routine to prevent extraneous values from existing in a pcols-dimensioned array.
    - Put `!REMOVECAM/!REMOVECAM_END` labels around these initializations as any which remain after CAM is retired no longer need this precautionary step.
1. Repeat #1-4 with any routines which are called internally
1. Search through the code and make sure that all locations which call these updated routines have been modified correctly
    - Use the subsetted 1:ncol arrays when calling the updated routine 
    - Initialize all intent(out) arrays to zero before making the call

## 1d - Use kind_phys instead of r8
1. Remove `use shr_kind_mod, only: r8=> shr_kind_r8`
1. Replace with `use ccpp_kinds, only: kind_phys`
1. Do a find and replace for `r8` with `kind_phys` in the module (and any dependencies)

## 1e - Remove use statements
Remove all “use” statements and have the data appear in the calling list (the input and output variables to the routine).  The only exception to this rule is for use statements to routines contained within the parameterization package or dependencies.

- If a use statement is bringing in a variable from another module (e.g. something from `physconst`), add the variable to the calling list of the subroutine(s) that use it (and remove the use statement)
- If a use statement is bringing in an external routine (not part of the parameterization package), you have a few options (consult with the other CAM SEs on how to proceed):
    1. If the routine already exists in the core CAM-SIMA code tree (not in `ncar_ccpp`), ask another CAM SE about how and where to call the routine within the CAM-SIMA run loop and set a variable to be passed into the physics (variable will need to be added to the registry)
    1. CCPP-ize the external routine and/or module and add it to the suite definition file before or after the parameterization
    1. Postpone CCPP-zing the external routine and move the module to the `to_be_ccppized` directory in `atmospheric_physics` so it can be used as a dependency (we'll revisit this in [create metadata](create-metadata.md))

        !!! Note "'init' routine in to_be_ccppized module"
            If you are putting off ccpp-ization of a dependent module and that module has an init routine, you may have to add a call to that routine in the init phase of the `to_be_ccppized_temporary` scheme (`$atmospheric_physics/to_be_ccppized/to_be_ccppized_temporary.F90`) and ensure `to_be_ccppized_temporary` is included in your SDF

- Comment out `outfld`, `addfld` use statements for now
    - Also comment out the `addfld` and `outfld` calls within the module(s)

## 1f - Add error variables
Add `errmsg, errflg` to the end of your calling list.  Set `errflg` to non-zero if an error is encountered in your routine and set `errmsg` with an appropriate text indicating the error. 

The declarations for these variables are:
```
   character(len=512), intent(out) :: errmsg
   integer,            intent(out) :: errflg
```

At the top of your routine initialize the variables as follows:
```
 errmsg = ‘ ‘
 errflg = 0
```

## 1g - Replace state and ptend variables in calling list
`state` and `ptend` variables will not be passed in as full objects, but instead as the individual state and ptend fields that are used in the routine.

If `state` appears in the calling list:

1. Search the routine for "state%" and, for each of the distinct variables found (`state%<var>`), add the variable (`<var>`) to the calling list
1. Replace all calls to `state%<var>` with `<var>`
1. Remove `state` from the calling list

If `ptend` appers in the calling list:

1. Search the routine for "ptend%" and, for each of the distinct variables found (`ptend%<var>`), add the variable to the calling list
    - the variable name is up to you, some suggestions:
        1. `<var>_tend`
        1. `d<var>dt`
1. Replace all calls to `state%<var>` with the new name
1. Remove `ptend` from the calling list

## 1h - Remove pbuf variables
pbuf variables will not be in CCPP code. Instead, pass required individual variables as `intent(in)`, `intent(out)` or `intent(inout)` depending on whether they are being used or set in the interstitial.

For example, if the following line of code exists in a routine:

```
call pbuf_get_field(pbuf, taubljx_idx, taubljx)
```

You will want to search around in the routine (and routines that it calls) to determine if `taubljx` is read from, written to, or both:

1. If the variable is read from only (e.g. is only on the right-hand side of equations), the variable will be `intent(in)`
1. If the variable is written to only (e.g. is only on the left-hand side of equations), the variable will be `intent(out)`
1. If the variable is both read from and written to (e.g. it appears on both sides of equations), the variable will be `intent(inout)`

In this example, `taubljx` is `intent(out)`, so we'd remove the `pbuf_get_field` call and add `taubljx` to the calling list as an ouput variable.

## 1i - Mark variables as initialized
- If you add new variables that do not need to be read in from a file but come directly from the host model/registry and are uninitialized by CAM-SIMA, use the `mark_as_initialized` subroutine in the `<parameterization>_init` phase.
- Also, if you are setting a (non-namelist) public module level variable(a variable above the contains statement) in CAM-SIMA that will be used by a physics scheme, then also set the mark_as_initialized for that variable (also in the `<parameterization>_init` phase)

```
use phys_vars_init_check, only: mark_as_initialized
```
```
call mark_as_initialized('<standard name>')
```

## 1j - Initial standard name check
Do a preliminary look at the variables on the calling lists and make sure that CCPP standard names and units exist for all of them, by checking for them in [CAM Standard Names Spreadsheet](https://docs.google.com/spreadsheets/d/1vpQ_xDZk00Z-_3SpW5N2EF3_FY6K7opNN4cqtSMlbwU/edit?gid=0#gid=0).

- If variables are not filled out in the list, highlight the entire line with yellow.  
- If they don't exist at all in the spreadsheet, enter them into the bottom of the sheet and highlight them with yellow.  Note that physconst and diagnostic-only variables *may* not reside in the spreadsheet.
- Let Jesse, Courtney or Cheryl know if you've encountered missing CCPP standardnames and/or units and added them to the list, so they can be discussed at an upcoming meeting.  
- If you create your own name, add `_TBD` to the end of the name to signify that it has not been discussed yet.

!!! note
    This process can normally take 2-4 weeks, so a preliminary look is advisable.

## 1k - OPTIONAL: Make a CAM tag
Make ESCOMP/atmospheric_physics and ESCOMP/CAM tags if substantial changes have been made up to this point.  This is also a good step to take if there is potential for others to be making modifications to the same physics module.  This will allow them to make changes which can be merged via git commands. More on bringing your changes back to CAM [here](back-to-cam.md).

- Do at the very least a sanity compilation and run using the ESCOMP/CAM code base 
    - Delete the original module from src/physics/cam 
    - Use  your pulled apart modules in `atmos_phys/schemes/<parameterization_name>`.
    - In bld/configure in the ESCOMP/CAM source code, find the section "Add the CCPP'ized subdirectories".  Add following line:
```
print $fh "$camsrcdir/src/atmos_phys/schemes/<parameterization_name>\n";
```
- Modify code as needed until this modified code compiles and runs properly.
- If you want to benchmark your development to this point, you will need to open PRs to both ESCOMP/atmospheric_physics and ESCOMP/CAM.

Proceed to [2 - Create metadata](create-metadata.md)