# 2 - Create metadata

## Create template
To create a template metadata file based on the parameterization, run the following command in the directory with the routine you are converting (<parameterization>.F90):
```
python $CAM-SIMA/ccpp_framework/scripts/ccpp_fortran_to_metadata.py <parameterization>.F90 
```

If you get errors:

1. Confirm you included the [argtable lines](convert-portable-layer.md#1b-add-required-htmlinclude-lines) above each routine
1. If you see a message "Missing local variables" and the variable is there, it may be missing its intent attribute
1. Hopefully other errors will be easy to address; consult other CAM SEs if there's any remaining confusion/errors

Completion of `ccpp_fortran_to_metadata.py` will result in the creation of `<parameterization>.meta`

## Complete metadata

Replace all `enter_*` sections with appropriate information (standard_name, units, dimensions, long_name). Special handling for [error variables](#error-variables), [constituents](#constituents), and [dependencies](#dependencies) can be found below.

!!! Warning "Named dimension in metadata template"
    If you have a dimension in the metadata field which is NOT of the form `enter_standard_name_X:enter_standard_name_Y` but is rather a single dimension without a :, this means that you have a named dimension in your converted routine and you should remove the name and replace it with ":"

- *standard_name*: the mapping between CAM variables and standard_names can be found in [this spreadsheet](https://docs.google.com/spreadsheets/d/1vpQ_xDZk00Z-_3SpW5N2EF3_FY6K7opNN4cqtSMlbwU/edit?gid=0#gid=0). All official standard names (approved by all stakeholders) can be found in the [CCPPStandardNames repo](https://github.com/ESCOMP/CCPPStandardNames/blob/main/Metadata-standard-names.md). Note that this does not yet contain all of the spreadsheet names.
    - You may have to trace the variable back through the CAM code to find the `Snapshot or Local name` to look for
    - Officially accepted standard names can be found in this [repository](https://github.com/ESCOMP/CCPPStandardNames/blob/main/Metadata-standard-names.md), but the CAM Standard Names Spreadsheet should include all of the CAM variables.
    - If a standard name is not found:
        - CCPP names are trying to adhere to CF names, so you may find names on the CF web page: https://cfconventions.org/Data/cf-standard-names/76/build/cf-standard-name-table.html
        - If you need to create your own standard name, the current proposal for creating standard names can be found here: https://github.com/ESCOMP/CCPPStandardNames/blob/main/StandardNamesRules.rst 
        - May need to consult with WRF / MPAS scientists and other SIMA groups to coordinate StandardName usage.
        - Keep a list of StandardNames that you introduce and give them to a CAM SE for incorporation into an upcoming CCPPStandardNames PR
        - If you create your own name (for now), add `_TBD` to the end fo the name to signify that it has not been discussed yet
!!! Warning "Updated standard names"
    If, in the course of your implementation or during code review to the CCPPStandardNames repository, the standard name changes, please update the name in the spreadsheet to reflect the change!


- *units*: the units can be found in the standard names spreadsheet as well (`Snapshot Units` column)
- *dimensions*: Refer to the table below
    - If you have module level variables (variables above the "contains" line in a module file) which are declared with dimension `pver` or `pverp`, then you need to make this variable be allocatable and allocate the variable in the init routine.  This is due to the fact that pver is not known at compilation time, but is rather a run-time option in CAM-SIMA
    - Note for pcols inside parameterizations:  As only active columns will be passed to the parameterizations, there is no need for a maximum size variable anymore
    - In the template, you can convert `enter_standard_nameXX:enter_standard_nameYY` to just  use the appropriate dimension_name or do `1:dimension_name` (both work)

<div style="text-align:center" markdown>
| CAM dimension | CCPP phase | standard_name |
|:--------------|------------|---------------|
| pver          | all        | vertical_layer_dimension |
| pverp (or pver+1) | all    | vertical_interface_dimension |
| ncols or pcols| non-run phase | horizontal_dimension |
| ncols or pcols| run phase  | horizonal_loop_extent OR horizontal_loop_begin:horizontal_loop_end|
| pcnst         | all        | number_of_ccpp_constituents |
</div>

- *optional arguments*: If there are optional arguments, consult with a CAM SE. Optional attributes are an unsupported configuration, but are being incorporated into the framework. That said, there may be a workaround to avoid the use of optional arguments in your scheme
- Note: Try to avoid module level allocations and/or assignments even in the init phases of the code.  If you believe you need to do this, speak with a Jesse and/or Cheryl for guidance

Once you have created your metadata, proceed to [3 - Create namelist XML file](create-namelist-xml.md)

### Error variables
See below for error variable metadata
```
[ errmsg ]
  standard_name = ccpp_error_message
  long_name = Error message for error handling in CCPP
  units = none
  type = character | kind = len=512
  dimensions = ()
  intent = out
[ errflg ]
  standard_name = ccpp_error_code
  long_name = Error flag for error handling in CCPP
  units = 1
  type = integer
  dimensions = ()
  intent = out
```
### Constituents
See [constituent usage](../design/constituents.md/#constituent-usage). You can tell a variable is a constituent if it is in the `state%q` array in CAM (may have to trace back in the code a bit)

### Dependencies
If your scheme has any "use" statements (helper functions, approved dependencies), add a `dependencies` field with a common-separated list of relative paths to the necessary modules to the top of your metadata file. Example:
```
[ccpp-table-properties]
  name = musica_ccpp
  type = scheme
  dependencies = micm/musica_ccpp_micm.F90,musica_ccpp_util.F90
```