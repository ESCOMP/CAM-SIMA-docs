# 5 - Create an SDF
The **Suite Definition File (SDF)** tells the CCPP-Framework which schemes will be run in what order. For more, see [CCPP in CAM-SIMA](../design/ccpp-in-cam-sima.md)

- In `$CAM-SIMA/src/physics/ncar_ccpp/suites`, create `suite_<parameterization>.xml`. This is your SDF!
- See the template below. You will need to select either `physics_before_coupler` or `physics_after_coupler` as your group name.
```
<?xml version="1.0" encoding="UTF-8"?>

<suite name="<parameterization>" version="1.0">  
   <group name="physics_X_coupler">  
       <scheme><parameterization></scheme>
   </group>
</suite>
```

<div style="text-align:center" markdown>
| CAM physpkg routine | CAM-SIMA SDF Group     | Description |
|:--------------------|------------------------|-------------|
| tphysbc             | physics_before_coupler | Physics schemes run before the surface coupler is called |
| tphysac             | physics_after_coupler  | Physics schemes run after the surface coupler is called  |
</div>

- Add any interstitials which you’ve created or are using in the appropriate location relative to your parameterization

!!! Note "CAMX physics suite"
    You will also need to add your same set of schemes to the appropriate larger suite (usually `suite_cam7.xml`). Use comments to delineate your set(s) of schemes to make the SDF somewhat readable.

For example: if your parameterization requires dry water vapor, you'll want to add:

- `<scheme>wet_to_dry_water_vapor</scheme>` before your core parameterization scheme to convert water vapor to a dry mixing ratio (CAM-SIMA has "wet")
- `<scheme>dry_to_wet_water_vapor</scheme>` after your parameterization scheme to convert back to a wet mixing ratio to return to CAM-SIMA

Once you have created your SDF, proceed to [6 - Create snapshots of CAM](create-snapshots.md)