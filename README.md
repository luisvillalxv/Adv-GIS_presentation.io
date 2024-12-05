# Aquifer properties function
## Objetive
Develop a code that shows some of the relevant characteistics of an aquifer (water table altitude and vadose zone tickness) and create a geodata base that contains the layers with this information. As inputs for this code, the user needs to supply well points with the depth to water and a Digital Elevation Model (DEM). The output layers are intended to generate a quick visualization of the aquifer conditions, but the results should be verified because the code uses a generic interpolation model thac couldn't adjust in a valid way to the input data. For this reason, the results for this script also include a Standard Error Map that shows to the user the accuracy of the interpolation model.
## Relevance
Generally, the analysis of the conditions of an aquifer starts with the collection of data such as the depth to the water level. However, this information is difficult to interpret quickly because the topographic altitude is generally not linked to the altitude of the water level and it is necessary to perform spatial interpolation analysis to have an approximation of the conditions of the aquifer, which is usually time-consuming and requires resources. Therefore, a model that helps to visualize and generate data on the conditions of the aquifer quickly is very useful to generate deeper analysis or to know the areas in which a greater amount of data is required to be analyzed.
## Flow process

### Analyze the spatial reference for the input layers
- Review if the cordinate system for both layers is the same.
  - If the coordinate system is diferent ask to the user a EPSG code for reproject the layers and save this layers in the output geodatabase.


```python
# Analyze the spatial reference for the input layers and reproject if is needed.
desc_fc = arcpy.Describe(wellData)
desc_raster = arcpy.Raster(DEM)
if desc_fc.spatialReference.factoryCode != desc_raster.spatialReference.factoryCode:
    print(f"The coordinate system for the Well data points is: {desc_fc.spatialReference.name} ({desc_fc.spatialReference.factoryCode}) and the coordinate system for the DEM is: {desc_raster.spatialReference.name} ({desc_raster.spatialReference.factoryCode})")
    print("A reprojection is needed for develop this analysis")
    CS = int(input("Enter the EPSG code to project both layers: "))
    arcpy.management.Project(wellData,"AquiferProperties.gdb\\welldData_RP",arcpy.SpatialReference(CS))
    arcpy.management.ProjectRaster(DEM, "AquiferProperties.gdb\\DEM_RP", arcpy.SpatialReference(CS))
    wellData = "AquiferProperties.gdb\\welldData_RP"
    DEM = "AquiferProperties.gdb\\DEM_RP"
    print("-------------------------------------------------------------------------")
```
<p align="center">
Script 1. Analyze the spatial reference for the input layers and reproject it if is needed.
</p>

### Calculate the water table altitude for each well point
- List the fields in the well data points layer for select the field that contains the water depht.
- Add the altitude from the DEM to each well data point.
- Substract well altitude to the water depth to obtain the water table altitude.
```python
# List the fields in Well data points layer.
  fields = arcpy.ListFields(wellData)
  print(f"The field names in the layer {wellData} are:")
  
  # Print the name for each field in the layer.
  for field in fields:
      print(field.name, end = ", ")

  # Ask the user to enter the field that contains the well depth to water.
  wellDepthField = input("Enter the field that contains the well depth to water: ")

  # Extracts DEM values to the well data points to create a new feature class.
  arcpy.sa.ExtractValuesToPoints(wellData, DEM, "AquiferProperties.gdb\\welldData_v2")
  wellData2 = "AquiferProperties.gdb\\welldData_v2"
```
<p align="center">
Script 2. Calculate the water table altitude for each well point.
</p>

### Calculate the water table altitude surface and its standard error
The script uses a Epirical Bayesian Kriging interpolation model with a generic configuration for calculate the water table altitude surface and its standard error from the water table altitude points following the next logic:
- Set the local variables for the interpolation
- Set the neighbourhood search variables
- Calculate the interpolation prediction for the water table altitude with the Empirical Bayesian Kriging model
- Calculate the interpolation standard error

```python
# Set local variables for the interpolation.
inPointFeatures = "welldData_v2"
zField = "WaterAltitude"
outLayer = "outEBK_GA"
outRaster = "EBK_predict"
transformation = "NONE"
maxLocalPoints = 30
overlapFactor = 0.5
numberSemivariograms = 100

# Set variables for search neighborhood.
radius = 50000
smooth = 0.4
searchNeighbourhood = arcpy.SearchNeighborhoodSmoothCircular(radius, smooth)
outputType = "PREDICTION"
quantileValue = ""
thresholdType = ""
probabilityThreshold = ""
semivariogram = "POWER"

# Execute Empirical Bayesian Kriging for calculate the prediction.
arcpy.EmpiricalBayesianKriging_ga(inPointFeatures, zField, outLayer, outRaster,
                                  cellSize, transformation, maxLocalPoints, overlapFactor, numberSemivariograms,
                                  searchNeighbourhood, outputType, quantileValue, thresholdType, probabilityThreshold,
                                  semivariogram)

# Execute Empirical Bayesian Kriging for calculate the prediction standard error.
outRaster = "EBK_SE"
outputType = "PREDICTION_STANDARD_ERROR"
arcpy.EmpiricalBayesianKriging_ga(inPointFeatures, zField, outLayer, outRaster,
                                  cellSize, transformation, maxLocalPoints, overlapFactor, numberSemivariograms,
                                  searchNeighbourhood, outputType, quantileValue, thresholdType, probabilityThreshold,
                                  semivariogram)
```
<p align="center">
Script 3. Calculate the water table altitude surface and its standard error.
</p>

### Caluculate the Vadose Zone Tickness
- Substract the water table altitude surface to the DEM
