# Create database from BCA Data Advice (XML) and Inventory Extract (TXT) files
Overview steps (details in the sections below). Applies to annual Data Advice files (REVD) and corresponding Inventory Extracts). See the [BC Assessment data user guides](https://www1.bcassessment.ca/Support/Guide) for information about data fields.

1. Download data to a workstation capable of XSLT streaming (e.g. UBC Library Digital Scholarship Lab workstations with Oxygen software)
2. Use `sed` to add attributes to "AssessmentArea" and "Jurisdiction" elements in the XML file
3. Run XSL transformation on the modified XML file using "BCA_REVD_stylesheet.xsl" stylesheet
4. Merge and add headers to Residential and Commercial Inventory Extract files
5. Create a "metadata.csv" file with information about the source files and data
6. Create sqlite database from the output .csv files


## Step 1: download data
**UBC Library employees** 
If the files are not already in Abacus, authorized UBC Library employees may download them from the BC Assessment website:

- Login at https://bcassessment.ca using BCeID credentials. 
- Go to _Data Advice -> Files_ then select an annual .zip file for download (e.g YYYYMMDD_REVD20_0139.zip). Unzip on the workstation where you will transform the data.
- Go to _Reports -> Roll Reports_ and download one set of Commercial and Residential Inventory Extracts.

**UBC researchers**
Most UBC researchers can access the BC files from Abacus: 

- Go to the Abacus record (<>) and login with CWL credentials
- Download three files for the desired year: the REVD XML file, Commercial Inventory Extract, and Residential Inventory Extract. Unzip on the workstation where you will transform the data.

## Step 2: add attributes to selected XML elements in the Data Advice file (REVD)
In the XML data file the elements `AssessmentAreaCode`, `AssessmentAreaDescription`, `JurisdictionCode`, and `JurisdictionDescription` are not in the `FolioRecord` ancestor axis. That is a problem for XSL streaming since it's difficult to access this information without breaking streaming constraints. The workaround is to modify the source XML file before processing, adding `code` and `description` as attributes to `AssessmentArea` and `Jurisdiction` elements. No information is added or removed in the transformation but this...

```
<AssessmentArea>
  <AssessmentAreaCode>01</AssessmentAreaCode>
  <AssessmentAreaDescription>Capital</AssessmentAreaDescription>
  <Jurisdictions>
    <Jurisdiction>
      <JurisdictionCode>213</JurisdictionCode>
      <JurisdictionDescription>City of Colwood</JurisdictionDescription>
      ...
```

...becomes this:

```
<AssessmentArea code="01" description="Capital">
  <Jurisdictions>
    <Jurisdiction code="213" description="City of Colwood">
    ...
```
Use `sed` to make these changes and to replace all double quotes with single quotes (to avoid CSV output problems later). By default `sed` processes one line of the XML file at a time; the `N;N;` in the command below appends two lines to the context under review to allow for substitutions across three lines. This command may take 15 minutes to complete.

```
sed -i -e 's/"/'"'"'/g' -e '/<AssessmentArea>/{N;N;s/>\s*<AssessmentAreaCode>\([^<]*\).*\s*<AssessmentAreaDescription>\([^<]*\).*/ code="\1" description="\2">/g}' -e '/<Jurisdiction>/{N;N;s/>\s*<JurisdictionCode>\([^<]*\).*\s*<JurisdictionDescription>\([^<]*\).*/ code="\1" description="\2">/g}' name_of_data_file.xml
```

*Note:* this syntax works in gnu-sed (including Git Bash) but not OSX sed. On a Mac adjust the syntax or install gnu-sed (homebrew) and run the command using `gsed` instead of `sed`.

## Step 3: XSL transformation
Use a XSLT processor to apply the `BCA_REVD_stylesheet.xsl` to the XML data file, generating CSV files. The processor must be capable of XSL streaming to handle large BCA data files. The instructions below use the Saxon "Enterprise Edition" (EE) processor included with Oxygen, installed on UBC Library Digital Scholarship Lab workstations (available for walk-in use, or access remotely at <http://remotelabs.ubc.ca>). The `BCA_REVD_stylesheet.xsl` stylesheet is streaming-compliant using XSL version 3.

- Save `BCA_REVD_stylesheet.xsl` on the DS Lab workstation
- Open _Oxygen XML Editor_ and create a new project
- Create a new _Transformation Scenario_:
	- if the _Transformation Scenarios_ window is not visible on the right, go to _Window -> Show View -> Transformation Scenarios_
	- click the _+_ symbol to add a new _XML Transformation with XSLT_
	- In the _XSLT_ tab set *XML URL* to the source XML file (modified in step 2) and *XSL URL* to `BCA_REVD_stylesheet.xsl`. 
	- Change the _Transformer_ to _Saxon EE_ and click the _Advanced options_ icon next to the drop-down menu; in the new window place a checkmark next to _Enable streaming mode_, then click _OK_
	- In the *Output* tab enter a valid file path (output will be directed elsewhere by the stylesheet but this setting is required).
	- Name the scenario and click _OK_
- The new scenario will be listed in the _Transformation Scenarios_ window; place a checkmark next to it and click the play icon to _Apply associated scenarios_  

If there are problems with the transformation scenario settings, error messages will display near the bottom of the screen. If all goes well there will be a "Transformation in progress" message at the bottom of the window. The transformation will take about 15 minutes to finish and the CSV files will be saved to an `output` folder in the same directory as the `BCA_REVD_stylesheet.xsl` stylesheet.


## Step 4: add headers and rename Inventory Extract TXT files
Four times per year the BCA provides a series of .txt files with additional information for each roll number:
-`YYYYMMDD_Commercial_Inventory_Extract.txt` (one file for entire province)
-`YYYYMMDD_A##_Residential_Inventory_Extract.txt' (one file for each Assessment Area)`
These files are already in csv format but are missing headers.

### Residential inventory
Combine the individual residential inventory files into one file:
`cat *.txt > residentialInventory.csv`

Add a header row to the new file. For field names and descriptions see the [Residential Invenotry Extract user guide](https://www1.bcassessment.ca/Support/Guide). 

`sed -i '1i area,jurisdiction,roll_number,MB_manual_class,blank1,blank2,MB_year_built,MB_effective_year,MB_total_finished_area,MB_num_storeys,num_full_baths,num_3-piece_baths,num_2-piece_baths,num_bedrooms,num_dens,blank3,type_of_foundation,num_multi_garage,num_single_garage,num_carport,lc_code_1, lc_code_2,lc_code_3,lc_code_4,lc_code_5,lc_code_6,pool_flag,other_building_flag,land_metric_flag,land_width,land_depth,land_sq_measure,land_area,blank4,inc_unit_of_measure_value,inc_unit_of_measure_code,blank5,inc_floor_num,inc_effective_year,plumbing_nil_flag,basement_finish_area,basement_total_area,deck_sq_footage,deck_sq_footage_covered,blank6,fireplace_num_1,blank7,fireplace_num_2,blank8,fireplace_num_3,blank9,fireplace_num_4,blank10,fireplace_num_5,first_floor_area,second_floor_area,third_floor_area,school_district,zoning' residentialInventory.csv`

Copy the modified file to the output folder.

### Commercial inventory
Rename to 'commercialInventory.csv' then add a header row. See [Commercial Inventory Extract user guide](https://www1.bcassessment.ca/Support/Guide) for field names and descriptions. 

`sed -i '1i area,jurisdiction,roll_number,year_built,effective_year,number_of_storeys,gross_leasable_area,net_leasable_area,parking_type_underground,parking_type_surface,parking_type_parking_structure,units_in_co-op,bachelor_units_in_co-op,1-bedroom_units_in_co-op,2-bedroom_units_in_co-op,3-bedroom_units_in_co-op,4-bedroom_units_in_co-op,predominant_manual_class,gross_building_area,basement_area,strata_ic_lot_area,apartment_units,apartment_bachelor_units,apartment_1-bedroom_units,apartment_2-bedroom_units,apartment_3-bedroom_units,apartment_4-bedroom_units,apartment_house_keeping_rooms,hotel_units,motel_units,senior_housing_units,senior_housing_bachelor_units,senior_housing_1-bedroom_units,senior_housing_2-bedroom_units,senior_housing_3-bedroom_units,senior_housing_bed_sitting_rooms,senior_housing_licensed_care_private_beds,senior_housing_licensed_care_semi-private_beds,total_balcony_area,blank3,mezzanine_area,type_of_heating,blank4,elevators,type_of_construction,other_buildings,school_district,zoning' commercialInventory.csv`

Copy modified file to the output folder.

## Step 5: create "metadata.csv" file
To help distinguish among SQLite databases for different years, create a file named "metadata.csv" containing:

- filenames of the source data as downloaded from the BC Assessment website
- selected metadata elements and attributes from the Data Advice REVD XML file

Follow this format, replacing the second column with the correct values (example from REVD21 database):

```
field,value
RollYear,2021
OwnershipYear,2022
RunType,REVD
RunDate,2021-08-19
StartDate,2020-01-01
EndDate,2020-12-31
SourceREVDFilename,20210819_REVD21_0139.xml
SourceResidentialInventoryFilename,20210707_Residential_Inventory_Extract.zip
SourceCommercialInventoryFilename,20210701_Commercial_Inventory_Extract.txt
```

## Step 6: create sqlite database
The proposed method uses [csv-to-sqlite](https://pypi.org/project/csv-to-sqlite/) but there are other suitable approaches. Navigate to the output folder containing all 12 .csv files and enter the command below (if on a computer with csv-to-sqlite installed). The `-t none` option indicates that all fields should be imported as "text" (instead of csv-to-sqlite guessing at the data type). If this is not specified, fields like roll_number will be interpreted as integers and leading zeros will be omitted.

```
ls *.csv | csv-to-sqlite -t none -o REVDXX_and_inventory_extracts.sqlite3
```
