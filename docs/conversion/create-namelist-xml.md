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


<entry id="rayk0">
    <type>integer</type>
    <category>rayleigh_friction</category>
    <group>rayleigh_friction_nl</group>
    <standard_name>rayleigh_friction_center_vertical_index</standard_name>
    <units>1</units>
    <desc>
      Variable to specify the vertical index at which the
      Rayleigh friction term is centered (the peak value).
    </desc>
    <values>
      <value>2</value>
    </values>
</entry>


</entry_id_pg>
```
!!!Note "namelist values"
    When populating the value fields, you don't need to port all of them at this time.  You need to put in the value that is used by default by CAM, but the rest will be ported once all schemes have been ported.

Once you have made your namelist file, proceed to [4 - Interstitials](interstitials.md)