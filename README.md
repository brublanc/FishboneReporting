# <u>Introduction</u>
A fishbone is the visual representation of where an address point actually is and where the system thinks it should be based on the ranges of a corresponding road centerline. This tool creates such fishbones using FME. For a more detailed explanation of the logic used in this tool, refer to the Technical Details section below.


# <u>Requirements</u>
* FME 2025.2 or newer (older versions may work as well, but are not guaranteet to)
  * ...or equivalent in Esri's Data Interoperability extension
	 

## Workflow-specific Requirements
### *Option 1: External geocoding (using a hosted locator):*
* Address point layer with the following fields (or their equivalents):
  * Full address
  * Zone
* [Street Address locator](https://pro.arcgis.com/en/pro-app/latest/help/data/geocoding/introduction-to-locator-roles.htm) or equivalent
  * Note: *Must support batch geocoding*
	* 
### *Option 2: Internal geocoding (using FME built-in logic):*
1. Address point layer with the following fields (or their equivalents). These fields are aligned with the [NENA GIS Data Model v3.0](https://www.nena.org/page/standards#NENA-STA-006) (section 4.2):
	* Add_Number
	* St_PreMod
	* St_PreDir
	* St_PreTyp
	* St_PreSep
	* St_Name
	* St_PosTyp
	* St_PosDir
	* St_PosMod
	* A1
	* A2
	* A3
	* A4
2. Road Centerline layer that corresponds to the same area as the address points. Expecting the following fields (or their equivalents). These fields are aligned with the [NENA GIS Data Model v3.0](https://www.nena.org/page/standards#NENA-STA-006) (section 4.1): .
	* FromAddr_L
	* ToAddr_L
	* FromAddr_R
	* ToAddr_R
	* St_PreMod
	* St_PreDir
	* St_PreTyp
	* St_PreSep
	* St_Name
	* St_PosTyp
	* St_PosDir
	* St_PosMod
	* A1_L
	* A1_R
	* A2_L
	* A2_R
	* A3_L
	* A3_R
	* A4_L
	* A4_R


# <u>Quick Start</u>
### *Option 1: External geocoding (using a hosted locator):*
1. Set up a reader to import address points
2. Set up street address locator or equivalent. Using an ArcGIS Enterprise one by default, but others would work as well
3. If necessary, reproject the source address points
4. Take note of the projection that's being used, specifically the units that the projection uses to measure distances (meters, feet, etc.)
5. If necessary, adjust the length parameter (user parameter, but can be hardcoded as well). This is what determines when a fishbone is "too long." Use the units that the projection uses. This distance is set to 10,580.00 ft by default (2 miles) 
6. Set up a writer for the resulting fishbones (line features)
7. [Optional] Set up a writer for the resulting fishbone reports/errors (point features)
   * By default, this sends a point feature at the location of the address point for each feature that failed to geocode

### *Option 2: Internal geocoding (using FME built-in logic):*
1. Set up a reader to import address points
2. Set up a reader to import road centerlines
3. If necessary, reproject source layers
4. Take note of the projection that's being used, specifically the units that the projection uses to measure distances (meters, feet, etc.)
5. If necessary, adjust the length parameter (user parameter, but can be hardcoded as well). This is what determines when a fishbone is "too long." Use the units that the projection uses. This distance is set to 10,580.00 ft by default (2 miles) 
6. Set up a writer for the resulting fishbones (line features)
7. [Optional] Set up a writer for the resulting fishbone reports/errors (point features)
   * By default, this sends a point feature at the location of the address point for each feature that failed to geocode


# <u>Technical Details</u>
### *Option 1: External geocoding (using a hosted locator):*
To create the fishbones, FME first imports the address point layer and creates an "_AddressZone" attribute (field) that's formatted in the following way [address number] [full street name] [zone]. An example would be "100 S Main St Memphis." The state or provice is not necesary. The zone field can be anything that differentiates the same address existing in different areas. This is typically the city/community, but it can also be postal codes or even neighborhood names if they exist.

The next step is to set up a locator that these address points will run against. This locator can be anything as long it covers the same data that your address points represent. An important factor to consider here is cost. Self-hosted and open source locators exist, but many commercial sources require api keys that may or may not have a cost associated with them. Of course this depends on the amount of requests being made, but it is important to be mindful of this. Additionally, if using a self-hosted locator, make sure that batch geocoding is supported, especially if sending a large number of requests.  Locators that do not support batch geocoding may take a long time to run through your data.

After geocoding the incoming addresses, the results are split into two data streams: one that exports the resulting fishbones, and one that exports any errors that occurred during the geocoding process (timeouts, failed to geocode, etc.).

The results data stream proceeds to then draw a line between the original address point and the geocoded location. This is done through a [LineBuilder](https://docs.safe.com/fme/html/FME-Form-Documentation/FME-Transformers/Transformers/linebuilder.htm). The length of the line is then used to filter out any fishbones that are deemed to be too long (2 miles by default). If not too long, export the fishbones to the writer, otherwise remove the last vertex (on the geocoded location) and send to be exported as a fishbone error along side the locator fishbone errors.

Fishbone errors are given some attribution to help identify the issues, then exported to the appropriate fishbone errors writer.

### *Option 2: Internal geocoding (using FME built-in logic):*
To create the fishbones, FME first imports the address point layer and the road centerline layer. Each data stream is processed separately according to the details listed below:

#### <u>Address points</u>:
After the points are imported and potentially reprojected, a matching field called "_matchlabel_withzone" is created in-memory. This field is a concatenation of the following fields (spaces or equivalent values don't matter here): Add_Number + (street name components*) + (APolygon components**). An example of a resulting value for this field is "3150-----Lenox Park-Blvd---TN-Shelby-Memphis-". In this example, dashes were used between elements; if the element was not present for that address, it was left empty (hence all the ---).

This concatenated field is then passed downstream to be matched with their corresponing position in the road centerlines (see below).

#### <u>Road Centerlines</u>:

After the exceptions are filtered out, and road centerlines are imported & potentially reprojected, the following fields are created in-memory:
* _seglenth: the length of the road segment
* _leftrange: the range of address numbers on the left side of the road segment (calculated as ToAddr_L - FromAddr_L)
* _rightrange: the range of address numbers on the right side of the road segment (calculated as ToAddr_R - FromAddr_R)
* _leftincrement: ((_seglength)/2 + 1) for odd ranges or ((_seglength)/2) for even ranges. These are used to determine how far along the road centerline the geocoded point should be placed based on its address number. More details can be found within the scripts themselves
* _rightincrement: ((_seglength)/2 + 1) for odd ranges or ((_seglength)/2) for even ranges. These are used to determine how far along the road centerline the geocoded point should be placed based on its address number. More details can be found within the scripts themselves
* _intervallength_l: the length of the interval for the left side of the road segment, calculated as (_seglength)/(_leftincrement)
* _inervallength_r: the length of the interval for the right side of the road segment, calculated as (_seglength)/(_rightincrement)
* _countinglabel_l: a concatenated field that includes the street name components<sup>1</sup> and APolygon components<sup>2</sup> for the left side of the road segment. This is used for counts that are needed downstream
* _countinglabel_r: a concatenated field that includes the street name components<sup>1</sup> and APolygon components<sup>2</sup> for the right side of the road segment. This is used for counts that are needed downstream
* _FullName: a concatenated field that includes the street name components* for both sides of the road segment. This is used for matching with the address points downstream

The next step is to split the road centerline segments into smaller segments based on the interval lengths that were just calculated for each side. This is done via a [Chopper](https://docs.safe.com/fme/html/FME-Form-Documentation/FME-Transformers/Transformers/chopper.htm?Highlight=chopper) with group processing using the _intervallength fields. A [Densifier](https://docs.safe.com/fme/html/FME-Form-Documentation/FME-Transformers/Transformers/densifier.htm) is used prior to this per Safe's suggestion.

We then use the _countlabel fields get the sequential order of each chopped segment in relation to its original parent segment. The resulting _count1 attribute starts at an index of 1. We repeat this step for a second count attribute (_count2) that starts at -1 instead. We add both of these to the FromAddr_L attribute to get an accurate _leftnumber that represents the geocoded location along the line. For a full example illustrating this, see the Examples section.

We also replace the linear geometry of the chopped road centerline segments with point geometries at the starting node of each segment. This will serve as the ending node for each fishbone (with the address points themselves being the starting nodes).

A [duplicate filter](https://docs.safe.com/fme/html/FME-Form-Documentation/FME-Transformers/Transformers/duplicatefilter.htm) is used at this stage to remove any duplicate points that may have been created during the chopping process. This is important to ensure that the same point is not matched with multiple address points downstream.

It is at this point can do an attribute join between the chopped segments and the address points using a matching field (_matchlabel_withzone). This field is already complete for the address points, but it needs to be calculated for the chopped segment points by concatenating the recently calculated range value and the existing _FullName field. The formatting of these fields needs to be exactly the same for the address points and the chopped road centerline segments for the join to work properly. <span style="color: red;">This is a critical step, as it is where the geocoding happens in this workflow</span>.

The last step is to use a [Line Builder](https://docs.safe.com/fme/html/FME-Form-Documentation/FME-Transformers/Transformers/linebuilder.htm) to connect the starting node (address point) with the ending node (chopped road centerline point) to create the fishbone lines. The length of the resulting line is then used to filter out any fishbones that are deemed to be too long (2 miles by default). If not too long, export the fishbones to the writer, otherwise remove the last vertex (on the geocoded location) and send to be exported as a fishbone error along side any other errors that may have been created during the process.

# <u>Notes</u>
<sup>1</sup>Street name components include the following fields: *St_PreMod, St_PreDir, St_PreTyp, St_PreSep, St_Name, St_PosTyp, St_PosDir, St_PosMod*

<sup>2</sup>APolygon components include the following fields: A1, A2, A3, A4 (or their _L and _R equivalents in the road centerline layer)

### <u>Additional Notes</u>
* Not using A5 fields (equivalent of neighborhoods) because they are not commonly used in many datasets. But they can be added as needed as well.
* Using the [NENA GIS Data Model v3.0](https://www.nena.org/page/standards#NENA-STA-006) schema, but local schemas can work as well.
* While this tool has been somewhat optimized for performance, it may still be slow to run through large datasets. This is especially true for the internal geocoding option, which relies on FME's built-in logic to determine the geocoded location along the road centerline. The external geocoding option may be faster, but it also depends on the performance of the locator being used and the number of requests being made.

### <u>Examples</u>

Assuming that we have a segment with the following attributes:
* <span style="color:green;">_seglength</span> = <span style="color:orange;">50</span> ft
* <span style="color:green;">FromAddr_L</span> = <span style="color:orange;">120</span>
* <span style="color:green;">ToAddr_L </span> = <span style="color:orange;">150</span>

We can deduce the following:
* <span style="color:green;">_leftrange</span> = <span style="color:orange;">30</span> (to - from)
* <span style="color:green;">_leftincrement</span> = <span style="color:orange;">16</span> ((_leftrange/2) + 1)
* <span style="color:green;">_intervallength_l</span> = <span style="color:orange;">1.875</span> (_seglength/_leftincrement)

... meaning that the original segment is split into <span style="color:yellow;">16 smaller segments that are 1.875 ft each<span>.

Next we calculate the part of the original range that each segment needs to have. Note that in this example, we are working with even values on the left side.

First, we create an indexed list of where each segment falls within the original segment by using a [Counter](https://docs.safe.com/fme/html/FME-Form-Documentation/FME-Transformers/Transformers/counter.htm). The index for this starts at 1 (segment 1 = 1, segment 2 = 2, etc.).

We repeat this step, but this time the index count starts at -1 (segment 1 = -1, segment 2 = 0, etc.).

We then add both of these values to the original FromAddr_L of this segment (120 in this case). Leaving us with the following matrix:

| FromAddr_L | _count1 | _count2 | sum*|
|------------|---------|---------|-----|
|    120     |    1    |   -1    | 120 |
|    120     |    2    |    0    | 122 |
|    120     |    3    |    1    | 124 |
|    120     |    4    |    2    | 126 |
|    120     |    5    |    3    | 128 |
|    ...     |   ....  |   ...   | ... |
|    120     |    14   |   12    | 146 |
|    120     |    15   |   13    | 148 |
|    120     |    16   |   14    | 150 |

*This is the value that is used for matching with the address points. For example, if we have an address point with an Add_Number of 126, it would match with the segment that has a sum of 126 in this matrix. 


# <u>Future Work</u>
1. Host this tool on FME Hub for easier access and usability. This would allow users to easily download and use the tool without having to go through the process of setting it up themselves. It would also allow for easier updates and maintenance of the tool in the future.
2. Add an open-source python version for in-memory geocoding (future option 3).
3. Provide testing data in the repository for users to test the tool and understand how it works before using it with their own data. This would also allow for easier troubleshooting and debugging of the tool if users encounter any issues. 
4. Add a configurable offset to the geocoded point along the road centerline to align with existing industry standards for geocoding.