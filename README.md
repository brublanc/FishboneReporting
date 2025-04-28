# Introduction

A fishbone is the visual representation of where an address point actually is and where the system thinks it should be based on the ranges of a corresponding road centerline. This tool creates such fishbones using FME. For a more detailed explanation of the logic used in this tool, refer to the Technical Details section below.

# Requirements
* FME 2024.2 or newer (older versions may work as well, but are not guaranteet to)
  *...or equivalent in Esri's Data Interoperability extension
* Address point layer with the following fields (or their equivalents):
  * FUll address
  * Zone
* [Street Address locator](https://pro.arcgis.com/en/pro-app/latest/help/data/geocoding/introduction-to-locator-roles.htm) or equivalent

# Quick Start
1. Set up a reader to import address points
2. Set up street address locator or equivalent. Using an ArcGIS Enterprise one by default, but others would work as well
3. If necessary, reproject the source address points
4. Take note of the projection being used, specifically the units that the projection uses to measure distances (meters, feet, etc.)
5. If necesary, adjust the length parameter (user parameter, but can be hardcoded as well). This is what determines when a fishbone is "too long." Use the units that the projection uses. This distance is set to 10,580.00 ft by default (2 miles) 
6. Set up a writer for the resulting fishbones (line features)
7. [Optional] Set up a writer for the resulting fishbone reports/errors (point features)
   * By default, this sends a point feature at the location of the address point for each feature that failed to geocode
  
# Technical Details

To create the fishbones, FME first imports the address point layer and creates an "_AddressZone" attribute (field) that's formatted in the following way [address number] [full street name] [zone]. An example would be "100 S Main St Memphis." The state or provice is not necesary. The zone field can be anything that differentiates the same address existing in different areas. This is typically the city/community, but it can also be zip codes or even neighborhood names if they exist.

The next step is to set up a locator that these address points will run against. This locator can be anything as long it covers the same data that your address points represent. An important factor to consider here is cost. Self-hosted and open source locators exist, but many require api keys that may or may not have a cost associated with them. Of course this depends on the amount of request being made, but it is important to be mindful of this. Additionally, if using a self-hosted locator, make sure that batch geocoding is supported, especially if sending a large number of requests.  Locators that do not support batch geocoding may take a long time to run through your data.

After geocoding the incoming addresses, the results are split into two data streams: one that exports the resuling fishbones, and one that exports any errors that occurred during the geocoding process (timeouts, failed to geocode, etc.).

The results data stream proceeds to then draw a line between the original address point and the geocoded location. This is done through a [LineBuilder](https://docs.safe.com/fme/html/FME-Form-Documentation/FME-Transformers/Transformers/linebuilder.htm). The length of the line is then used to filter out any fishbones that are deemed to be too long (2 miles by default). If not too long, export the fishbones to the writer, otherwise remove the last vertex (on the geocoded location) and send to be exported as a fishbone error along side the locator fishbone errors.

Fishbone errors are given some attribution to help identify the issue, then exported to the appropriate fishbone errors writer.
