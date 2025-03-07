# History & model output

## Configuring the namelist
The syntax for updating a the configuration for a history file is:
```
<namelist_option>;<volume>: <setting>
```

Possible namelist options for your `user_nl_cam` (fields in <span style="color:lightgrey">grey</span> have not been fully implemented):

- `hist_add_inst_fields`: add instantaneous fields to a specified history file

<ul style="color:lightgrey">
<li>hist_add_avg_fields: add average fields to a specified history file</li>
<li>hist_add_min_fields: add minimum fields to a specified history file</li>
<li>hist_add_max_fields: add maximum fields to a specified history file</li>
<li>hist_add_var_fields: add standard deviation fields to a specified history file</li>
</ul>

- `hist_remove_fields`: remove a given field from a specified history file
- `hist_file_type`: type of file (options are "history", <span style="color:lightgrey">"satellite"</span>, and <span style="color:lightgrey">"initial_value"</span>) - defaults to "history"
- `hist_max_frames`: maximum number of samples written to a specified history file (after which a new one will be created)
- `hist_output_frequency`: frequency with which to write samples to a specified history file (syntax is `<integer>*<multiplier>` like `3*nhours`)
- `hist_precision`: precision of the specified history file (options are "REAL32" and "REAL64") - defaults to "REAL32"
- `hist_write_nstep0`: logical for whether or not to write the nstep = 0 sample (defaults to .false.)
- `hist_filename_template`: template filename for the specified history file
    - Defaults to "%c.cam.%u.%y-%m-%d-%s.nc" where "%c" is the case name, "%u" is the volume, "%y" is the year, "%m" is the month, "%d" is the day, and "%s" is the number of seconds since midnight GMT, with the timestamp itself representing the model time when the file is created.

**Example**
Take the following sample `user_nl_cam`:
```
hist_output_frequency;h1: 5*ndays
hist_max_frames;h1: 3
hist_add_inst_fields;h1: U
hist_add_inst_fields;h1: V, Q
hist_precision;h1: REAL64
hist_filename_spec;h1: my-history-file%m-%d
hist_write_nstep0;h1: .false.
```

It will be parsed by `hist_config.py` and this will be the relevant section of atm_in:
```
&hist_config_arrays_nl
    hist_num_inst_fields = 3
    hist_num_avg_fields = 2
    hist_num_min_fields = 0
    hist_num_max_fields = 0
    hist_num_var_fields = 0
/

&hist_file_config_nl
    hist_volume = 'h0'
    hist_avg_fields = 'T', 'Q'
    hist_max_frames = 1
    hist_output_frequency = '1*month'
    hist_precision = 'REAL32'
    hist_file_type = 'history'
    hist_filename_spec = '%c.cam.%u.%y-%m-%d-%s.nc'
    hist_write_nstep0 = .false.
/

&hist_file_config_nl
    hist_volume = 'h1'
    hist_inst_fields = 'U', ‘V’, ‘Q’
    hist_max_frames = 3
    hist_output_frequency = '5*ndays'
    hist_precision = 'REAL64'
    hist_file_type = 'history'
    hist_filename_spec = 'my-history-file%m-%d'
    hist_write_nstep0 = .false.
/
```

In plain English, a one-month run with these history configuration will result in a total of three files that will look something like these:

- my-history-file01-06.nc
    - This file will contain instantaneous output for U, V, and Q (eastward_wind, northward_wind, and water vapor)
    - It will contain three frames, one at each of the following times:
        - 0001-01-06 (time=5)
        - 0001-01-11 (time=10)
        - 0001-01-16 (time=15)
- my-history-file01-21.nc
    - This file will contain instantaneous output for U, V, and Q (eastward_wind, northward_wind, and water vapor)
    - It will contain three frames, one at each of the following times:
        - 0001-01-21 (time=20)
        - 0001-01-26 (time=25)
        - 0001-01-31 (time=30)
- <case-name>.cam.h0a.0001-02-01-00000.nc
    - This file will contain averaged output for T and Q (air_temperature and water vapor)
    - It will have one frame with the time calculated at the midpoint of the month

## Adding a diagnostic field to the CAM-SIMA source code
During **init** time, fields can be added to the possible field list with a call to `history_add_field`:

*history_add_field(diagnostic_name, description, vdim_name, avgflag, units, gridname, flag_xyfill, mixing_ratio)*

| Field                | Optional? | Type      | Description                                    |
|:---------------------|-----------|-----------|------------------------------------------------|
| diagnostic_name      | No        | string    | diagnostic name for the field - will be the name in netcdf output file |
| description          | No        | string    | Description of the variable                    |
| vdim_name            | No        | string    | vertical dimension: `horiz_only` for no vertical dimension (see note below this table); 'lev' for vertical_layer_dimension; 'ilev' for vertical_interface_dimension|
| avgflag              | No        | string    | default average flag; options: 'avg', 'inst', 'min', 'max', 'var' (standard deviation) |
| units                | No        | string    | variable units                                 |
| gridname             | Yes       | string    | gridname on which the variable's data is mapped (defaults to the physics grid) |
| flag_xyfill          | Yes       | string    | fill value for variable values                 |
| mixing_ratio         | Yes       | string    | constituent mixing ratio type ('wet' or 'dry'); not set to anything if not passed in |

!!!Note "horiz_only"
    `horiz_only` is a variable, not a string. If you have a variable with only a horizontal dimension, include this use statement in your module: `use cam_history_support, only: horiz_only`


Example:
```
call history_add_field('Q', 'water vapor mixing ratio', 'lev', 'avg', 'kg kg-1', mixing_ratio='wet')
```

It's important to avoid adding calls to `history_add_field` to the CCPP-ized physics schemes (to keep them portable). Instead, create a new diagnostics scheme and place that in the `cam_diagnostics` folder of the atmospheric_physics repository. The `history_add_field` call will be in the `init` phase.

## Outputting a diagnostic field to the CAM-SIMA source code
**After init time**, a variable's current values can be captured for output with a call to `history_out_field`:

*history_out_field(diagnostic_name, field_values)*

| Field                | Optional? | Type       | Description                                   |
|:---------------------|-----------|------------|-----------------------------------------------|
| diagnostic_name      | No        | string     | diagnostic name for the field - will cause an error if the diagnostic name was not added via history_add_field |
| field_values         | No        | real       | variable values to be stored in the history buffer(s) |

Example:
```
call history_out_field('Q', const_array(:,:,const_idx))
```

It's important to avoid adding calls to `history_add_field` to the CCPP-ized physics schemes (to keep them portable). Instead, create a new diagnostics scheme and place that in the `cam_diagnostics` folder of the atmospheric_physics repository. The `history_out_field` call(s) will likely be in the `run` phase.

## Using the output
The output files can be found in your `run` directory. They are in netCDF format. 

See the [ADF](https://github.com/NCAR/ADF) for lots that you can do with your results!

A few useful commands for inspecting netCDF data:

- To output the header/metadata information for the file (includes list of variables on the file):
```
ncdump -h file.nc
```

- To output the data for a specific variable on the file:
```
ncdump -v <varname> file.nc
```

- To compare two files*:

```
cprnc <file1_path> <file2_path>
```
- To get a slice of one file (subset of time samples):
```
ncks -F -d time,<start time>,<end time>,<step> <snapshot_file> <output_file_name>
```
Example to get time samples 3-5 from a file onto a new file called "split-file.nc":
```
ncks -F -d time,3,5,1 file.nc split-file.nc
```

- To diff a variable between two files (and output the results to a new netcdf file):
```
ncdiff -v <field_name> -o <output_file> <file1> <file2>
```

*`cprnc` can be found:

- on derecho: `/glade/campaign/cesm/cesmdata/cprnc/cprnc`
- on izumi: `/fs/cgd/csm/tools/cime/tools/cprnc/cprnc`