# 4 - Interstitials
## Overview
An **interstitial** is a scheme that does calculations or variable modifications to connect what the host model has to what the scheme requires and vice versa.

- These schemes are placed before or after the core parameterization scheme in the suite definition file (SDF) to prep for the scheme or take what the scheme produces and translate it back to what the host model (CAM-SIMA) expects
- In CAM, interstitial code will appear as either function calls or calculations/modifications in the CAM interface just before or just after the call to the parameterization
    - If the interstitial code is a function call, that function/module should be CCPP-ized (if not already) and that scheme will be included in the SDF
    - It the interstitial code is loose in the CAM interface, it is recommended that, when possible, that code be moved into the scheme (either the beginning or end of the scheme subroutine)
        - UNLESS: it's a common (across multiple parameterizations) calculation/translation/modification. In which case, a new scheme should be created to do that calculation (scheme will live in `$CAM-SIMA/src/physics/ncar_ccpp/schemes/utilities`)
        - If it is absolutely necessary to create a scheme-specific interstitial, that scheme should be called `<parameterization>_pre` or `_post` (depending on where in the SDF it will be placed) and will live in the `ncar_ccpp/schemes/<parameterization>` directory

This section covers a few interstitial scenarios you are likely to face.

## Diagnostics interstitial
All `addfld`/`outfld` calls will go in a `<parameterization>_diagnostic.F90` interstitial. You can find a template and instructions for scheme-specific diagnostics [here](https://github.com/ESCOMP/atmospheric_physics/blob/main/schemes/sima_diagnostics/scheme_diagnostics_template.F90):

- `addfld` calls will go into the `<parameterization>_diagnostic_init` subroutine
- `outfld` calls will go into the `<parameterization>_diagnostic_run` subroutine
- The interstitial itself will reside after the `<parameterization>` line in the SDF
    - See: [History Usage](../usage/history.md/#adding-a-diagnostic-field-to-the-cam-sima-source-code) for the specifications of the CAM-SIMA-versions of `addfld` and `outfld` calls

## Utilities
As mentioned, there are some calculations/conversions/translations that are performed often throughout the physics code in CAM. These schemes are available for use in the `$CAM-SIMA/src/physics/ncar_ccpp/schemes/utilities` directory and include:

- **state_converters.F90**: contains common conversions/calculations of [state variables](../design/ccpp-in-cam-sima.md/#state-and-tendency-variables), including these schemes:

| Scheme name | Description | Output variable | Input variables |  
|:------------|-------------|-----------------|-----------------|
| temp_to_potential_temp | convert temperature to potential temperature | air_potential_temperature | air_temperature<br/>inverse_exner_function |
| potential_temp_to_temp | convert potential temperature to temperature | air_temperature |air_potential_temperature<br/>inverse_exner_function |
| calc_dry_air_ideal_gas_density |  Calculate density from equation of state/ideal gas law | dry_air_density |composition_dependent_gas_constant_of_dry_air<br/>air_pressure_of_dry_air<br/>air_temperature |
| calc_exner | calculate dimensionless exner function |  dimensionless_exner_function |composition_dependent_specific_heat_of_dry_air_at_constant_pressure<br/>composition_dependent_gas_constant_of_dry_air<br/>surface_reference_pressure<br/>air_pressure |
| wet_to_dry_water_vapor | convert water vapor from wet to dry mixing ratio |  water_vapor_mixing_ratio_wrt_dry_air |air_pressure_thickness<br/>air_pressure_thickness_of_dry_air<br/>water_vapor_mixing_ratio_wrt_moist_air_and_condensed_water |
| dry_to_wet_water_vapor | convert water vapor from dry to wet mixing ratio | water_vapor_mixing_ratio_wrt_moist_air_and_condensed_water | air_pressure_thickness<br/>air_pressure_thickness_of_dry_air<br/>water_vapor_mixing_ratio_wrt_dry_air |
| wet_to_dry_cloud_liquid_water | convert cloud liquid from wet to dry mixing ratio | cloud_liquid_water_mixing_ratio_wrt_dry_air | air_pressure_thickness<br/>air_pressure_thickness_of_dry_air<br/>cloud_liquid_water_mixing_ratio_wrt_moist_air_and_condensed_water |
| dry_to_wet_cloud_liquid_water | convert cloud liquid from dry to wet mixing ratio | cloud_liquid_water_mixing_ratio_wrt_moist_air_and_condensed_water | air_pressure_thickness<br/>air_pressure_thickness_of_dry_air<br/>cloud_liquid_water_mixing_ratio_wrt_dry_air |
| wet_to_dry_rain | convert rain from wet to dry mixing ratio | rain_mixing_ratio_wrt_dry_air | air_pressure_thickness<br/>air_pressure_thickness_of_dry_air<br/>rain_mixing_ratio_wrt_moist_air_and_condensed_water |
| dry_to_wet_rain | convert rain from dry to wet mixing ratio |  rain_mixing_ratio_wrt_moist_air_and_condensed_water |air_pressure_thickness<br/>air_pressure_thickness_of_dry_air<br/>rain_mixing_ratio_wrt_dry_air |

- **geopotential_temp.F90**: 
    - **geopotential_temp**: compute geopotential height (`geopotential_height_wrt_surface`) and geopotential height at interfaces (`geopotential_height_wrt_surface_at_interface`) from temperature (`air_temperature`)
- **static_energy.F90**
    - **update_dry_static_energy**: calculate dry static energy (`dry_static_energy`)
- **qneg.F90**:
    - **qneg**: Set values for constituent variables that are less than the minimum value to the minimum value (and print out what it's doing - configurable!)
    - You will want to include qneg in your SDF if you are modifying constituent tendencies in your scheme. It should be after the tendencies are applied and before `geopotential_temp`
    - If `qneg` is in your SDF, you will need to provide the output variable `scheme_name` in your physics parameterization that is modifying constituent tendencies

        !!! Warning "scheme_name variable"
            If you skip this step, you will get either `parse_source.CCPPError: Input argument for qneg_run, scheme_name, not found`, or an incorrect scheme_name from a previous routine will be used

- **physics_tendency_updaters.F90**: apply tendencies output by physics to state variables. You'll need to include a tendency updater in your SDF for any `ptend%X` variables in the CAM-version of your code.

| Scheme name | Description | inout variables | onput variables |
|:------------|-------------|-----------------|-----------------|
| apply_tendency_of_<br/>eastward_wind | Apply the eastward wind tendency calculated in the previous physics scheme(s) to the `eastward_wind` state variable | eastward_wind<br/>tendency_of_eastward_wind | tendency_of_eastward_wind_<br/>&ensp;&ensp;due_to_model_physics<br/>timestep_for_physics |
| apply_tendency_of_<br/>northward_wind| Apply the northward wind tendency calculated in the previous physics scheme(s) to the `northward_wind` state variable | northward_wind<br/>tendency_of_northward_wind | tendency_of_northward_wind_<br/>&ensp;&ensp;due_to_model_physics<br/>timestep_for_physics |
| apply_heating_rate | Apply the heating rate (`tendency_of_dry_air_enthalpy_at_constant_pressure`) to the temperature tendency and temperature state variable | air_temperature<br/>tendency_of_air_temperature_<br/>&ensp;&ensp;due_to_model_physics<br/>heating_rate | tendency_of_dry_air_enthalpy_<br/>&ensp;&ensp;at_constant_pressure<br/>composition_dependent_specific_heat_<br/>&ensp;&ensp;of_dry_air_at_constant_pressure<br/>timestep_for_physics |
| apply_tendency_of_<br/>air_temperature | Apply the temperature tendency calculated in the previous physics scheme(s) to the `air_temperature` state variable | air_temperature<br/>tendency_of_air_temperature | tendency_of_air_temperature_<br/>&ensp;&ensp;due_to_model_physics<br/>timestep_for_physics |
| apply_constituent_<br/>tendencies | Apply the constituent tendencies calculated in the previous physics scheme(s) to the constituent state array | ccpp_constituents<br/>ccpp_constituent_tendencies | timestep_for_physics |

Proceed to [5 - Create an SDF](create-sdf.md). 

!!! Warning
    You may have to revisit this step as you debug your code.