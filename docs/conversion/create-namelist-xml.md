# 3 - Create namelist XML file
If your scheme has a `readnl` scheme, create an .xml file with the same name as the scheme which uses the namelist (place it in the same location as the parameterization in `ncar_ccpp`.

- The filename should be `<scheme>_namelist.xml`
- If you have metadata entries for these namelist entries from your previous conversion steps, it may be useful to copy the metadata entry and edit it to be xml.  If you do this, you will delete the dimensions and intent lines and add category, group, desc and values/value.

The CCPP Framework will autogenerate the namelist reader based on the elements in the namelist XML file associated with the parameterization.  Each namelist element (`<entry_id>`) will have the following fields: `<id>` `<type>`, `<category>`, `<group>`, `<standard_name>`, `<units>`, `<desc>` and `<values>`.  Note that all variables in the namelist should appear in the parameterization’s `_init` subroutine interface. The description of each field is as follows:

- *id*: The namelist variable's name. The name must be lower case. The module converts all namelist variable names to lower case since Fortran is case insensitive.
- *type*:  An abbreviation of the fortran declaration for the variable. Any of these types may be followed by a comma separated list of integers enclosed in parentheses to indicate an array. The current namelist validation code only distinguishes between string and non-string types. Valid declarations are:
    - char*n
    - integer
    - logical
    - real
- *kind*: Only needed if type is real (if type is real, 'kind' will be 'kind_phys')
- *input_pathname* (optional): Only include this attribute to indicate that the variable contains the pathname of an input dataset that resides in the CESM inputdata directory tree. Note that the variables containing the names of restart files that are used in branch runs don't reside in the inputdata tree and should not be given this attribute. The recognized values are:
    - "abs" to indicate that an absolute pathname is required, or
    - "rel:var_name" to indicate that the pathname is relative and that the namelist variable "var_name" contains the absolute root directory
- *category*: A category assigned for organized the documentation. This can be found by copying from the `bld/namelist_files/namelist_definition.xml` file in CAM
- *group*: The namelist group that the variable is declared in (found in `<parameterization>_readnl` in CAM)
- *standard_name*: The CCPP standard name
- *units*: Units for the variable. Must adhere to CCPP units convention (link?)
- *desc*: The description for this namelist variable.  This can be found by copying from the `bld/namelist_files/namelist_definition.xml` file in CAM.  This is parsed and used in the creation of the CAM namelist web page.  Description should be adequate for this purpose (**Maybe not? This is implemented in SIMA yet...).
- *value*: Initial value for the variable. This can be a valid default value or an out-of-range value to allow for trapping in the code.
- *valid_values* (optional): Attribute that is mainly useful for variables that have only a small number of allowed values.

**Example Namelist XML file**
```
<?xml version="1.0"?>


<?xml-stylesheet type="text/xsl" href="namelist_definition.xsl"?>


<entry_id_pg version="2.0">


  <entry id="cldfrc_premit">
    <type>real</type>
    <category>cldfrc</category>
    <group>cldfrc_nl</group>
    <standard_name>tunable_parameter_for_top_pressure_bound_for_mid_level_clouds_for_cloud_fraction</standard_name>
    <units>Pa</units>
    <desc>
      Top pressure bound for mid level cloud.
    </desc>
    <values>
      <value dyn="mpas">75000.0D0</value>
      <value dyn="se">40000.0D0</value>
      <value>40000.0D0</value>
    </values>
  </entry>


</entry_id_pg>
```
!!!Note "namelist values"
    When porting namelist values from CAM namelist defaults, the attributes supported in CAM-SIMA can be retrieved by printing out `cam_nml_dict` (or `cam_nml_dict.keys()`) in `cime_config/buildnml` [at the end of the function call](https://github.com/ESCOMP/CAM-SIMA/blob/3699359ebfe81b00273a9815d128cae903a26208/cime_config/buildnml#L355). An example of supported attributes that may be commonly used are: `phys_suite`, `ic_ymd`, `dyn`, `hgrid`, `analytic_ic`, `ocn`.

## For physics schemes split into multiple CCPP schemes, have a dedicated namelist read scheme

For physics parameterizations that are split into multiple CCPP schemes (within a single or several modules), create a dedicated namelist options scheme to centralize namelist variable reading. This approach makes dependencies on namelist variables explicit, including in the case where a physics scheme depends on the namelist variables of another, different physics scheme (e.g., Hack shallow convection using ZM deep convection namelist options.) and facilitates future centralized modifications of namelist parameters for a physical scheme by scientists.

The namelist options scheme can follow the pattern of the [zm_conv_options](https://github.com/ESCOMP/atmospheric_physics/blob/main/schemes/zhang_mcfarlane/zm_conv_options.F90) scheme used for reading Zhang-McFarlane deep convective namelist options:
* The namelist options scheme **has to be in a separate module file**, e.g., `zm_conv_options.F90` and `zm_conv_options.meta`. No other CCPP schemes can be in the same module;
* Only a `init` phase is needed, e.g., `zm_conv_options_init`. Only having an `init` phase dedicated for namelist parameters, and no other compute, allows for these namelist variables to be read without having to run any logic for a particular scheme;
* The namelist XML file is attached to this module, e.g., `zm_conv_options_namelist.xml`.

Once this scheme is added to the SDF, namelist quantities defined in this namelist.xml are just regular CCPP quantities, i.e., they can be retrieved directly via their standard name as inputs to other physics schemes. There is no need to `use` variables from this namelist read module.

Once you have made your namelist file, proceed to [4 - Interstitials](interstitials.md)
