# Reqs for "DINA-Web archive" import API

This document includes some basic requirements needed to get started with the development of batch data import API to load data "archives" into the DINA-Web system. The idea is to start building a very simple service (like a prototype) that meets the needs of some specific use cases. 

## Edits 
Date | Version | Collaborator | Description
------------- | ------------- | ------------- | -------------
2015-02-12  |  1  | Markus E  | original author, provided idea and draft specification
2015-02-13  |  2  | Markus S  | some more suggestions and added scheme and minor edits here and there

Please just clone this repo and make changes to this document. If you are not part of the DINA-Web organization, send an message to get your github identity associated.

# Use cases

At a later stage, other use cases may be covered but currently the main use cases are:

1. The primary use case involves migration of some large botany datasets (including about 1 million specimen records). 
2. A secondary use case involves loading some Fishbase CC0 data originally provided in MS Access (.mdb) format into the DINA-Web system. This dataset is available for anyone to work with and can be found at https://github.com/DINA-Web/datasets/tree/master/fishbase.


## Background

Specify Workbench (Specify 6) is a tool that facilitates upload of data to the collection database. The tool can be used in many situations, but is unfortunately not suited for handling large datasets [1]. 

- This is for example reflected by the *low number of records (7,000)* that can be imported at a time.  
- Another limitation to Workbench is that it exposes *only parts of the Specify data model*, which restrict the user to a limited set of fields. 

These are the two major reasons why a large dataset import tool needs to be developed, besides the important life cycle management need to sunset the Specify 6 component in favor of web-friendly tools that are easier to deploy.

## Suggested approach

This is the suggested workflow (from left to right) to use when importing batches of data into the DINA-Web system.

```console

   /-------------\    /-------------\   /-------------\    /-------------\
   | Original    |    | Jailbroken  |   | Intermedi-  |    | Rest Web API|
   | datasource  |    | datasets    |   | ary batch   |    |             |
   | - "closed"  |    | - "open"    |   | - "DwC-A"-  |    |- write batch|
   | - format?   |>>>>| - exported  |>>>|   format    |>>>>|  units to   |
   | - hetero-   |    | - x-platform|   | - zipped    |    |  system     |
   |   genous    |    |   format    |   | - standard- |    |             |
   | - legacy    |    | - sqlite?   |   |   ized unit |    |             |
   \-------------/    \-------------/   \-------------/    \-------------/

   GENERAL SCENARIO FOR BATCH IMPORT USING DINA-WEB ARCHIVES................

   Current legacy       Converted into     Stored as         Fed to web api
   system exported      open formats       a DINA-Web        which rejects
   datasets in their    cleaned up and     compliant         with list of
   own proprietary      internally con-    batch archive     errors or loads
   format               sistent            ready to load     entire batch

   SPECIFIC EXAMPLE SCENARIO................................................

   Fishbase dataset      Converted to      Converted to       Posted to
   in .mdb / MS Access   sqlite3 file      DINA-Web batch     dina-web.net/api
   format                                  archive            batch import
                                                              endpoint                                                              
```                                                              

Some tools can be useful with this approach are: 

- legacy databases **export functions** (can FileMaker export in .csv?)
- alternatively open source tools to "jailbreak" proprietary formats by converting those into open formats such as sqlite3, mysql, postgres etc (such as **mdbtools** if you have access databases)
- open source tools such as **Open Refine** (http://openrefine.org/) to clean / prepare the open formatted data, do validation against external datasources etc (external consistency)
- open source **CLI tool** to create a "DINA-Web archive" and check it for internal consistency (can be done with R, Python, Java for example - ie packing up .csv files into a zip and verify its internal consistency)
- Web API **service endpoint** to use to upload the "DINA-Web archive" into a specific DINA-Web system (web api that receives posted archives)

## Details for botany use case

The use case described here involves migration of a large botany collection into the DINA-Web system. The data currently resides within a relational FileMaker Pro database, but can easily be exported to simple text-files (e.g. in a comma-delimited format). 

For now, only a part of the data is to be imported. All data to be imported are presumed to be clean and ready for import into the database – no validation should thus be necessary for the time being. 

### Backward compatibility constraints

Some data types, in particular geography, storage and taxon data types, aka "tree data" using Specify 6 terminology, are unfortunately required to be in a quite specific tree format due to some Specify 6 implementation details that make use of "node numbers" to store this data. For practical reasons, this type of data currently needs to be pre-loaded through the Specify 6 workbench before anything else is done. 

This means we are currently limited by a backward compatibility constraint, so we are not able to upload "tree data" directly along with other data type, because of technical reasons relating to retaining backward compatibility with the Specify 6 application which is still in use.

This also means that the import service needs to connect or link the uploaded data to already existing tree nodes and doesn't have to insert this data from scratch in the first iteration.

Note: an important design goal is to avoid dependencies on old Specify 6 components and code, as these are components that are not "web friendly" and that we want to sunset within a foreseeable future. So this constraint will have to be resolved in near future, somehow. Ideas?

### Typical user

The user of the import tool will be a data migration specialist. From the perspective of the user, the full procedure might look something like this:

1.  Prepare the data for import into Specify (extract and clean the data).
2.  Create a new collection using Specify Setup Wizard.
3.  Upload tree data (geography, storage and taxon) using Specify Workbench.
4.  Upload the rest of the data using the newly developed service.

## Data types

To start with, the import tool should be able to upload these different data types to the following 8 tables in Specify:  

- collectionobject
- preparation
- preptype
- collectingevent
- locality
- determination
- collector
- agent

To avoid dependencies on Specify 6 and preloading "tree data", at a later stage support is also needed for uploading these data types:

- geography
- storage
- taxon

## Using Darwin Core Archive as inspiration

For uploading data that already exist in a relational structure, a flat import format can be very inefficient. The Darwin Core Archive (DwC-A) format has been developed by GBIF in order to make it easier to exchange structured data [2]. 

Basically, a DwC-A is a zipped file-archive that comprises a number of text-files (in comma- or tab-delimited format) with information about different entities. One of the text-files (e.g. the one with specimen- or taxon-information) serves as the core and links to other 'extension' data files in a starlike manner.

## Suggested modifications of the DwC-A approach

A zipped file-archive would also be a suitable for uploading data into the DINA-Web system. However, a starlike schema would not suffice to represent the complexity in the data we would like to upload. Instead of having a single core with several extensions, the text-files in the archive should be related to each other in the way the corresponding tables are related in the database. 

To link individual records, the user should make use of universally unique identifiers in fields named 'GUID' in some of the tables. If an identifier is missing, it should be created by the migration specialist or by the data owner prior to migration. The uploaded identifiers can then be used at a later stage to unambiguously check imported records against their source.

## Requirement specification

The import tool should be able to:

1.  Upload data from a zipped file archive aka "DINA-Web batch import archive" with interlinked comma-delimited text-files.
2.  Connect uploaded data to existing tree nodes in the geography, storage and taxon trees.
3.  Automatically generate necessary metadata about individual records (e.g. Version and CollectionMemberID).

The format for data to be uploaded is described in Appendix A.

## References

[1] Specify Software Project. (17 August 2013). Importing External Data into Specify 6. [*http://specifyx.specifysoftware.org/wp-content/static/Importing%20External%20Data%20into%20Specify%206.pdf*](http://specifyx.specifysoftware.org/wp-content/static/Importing External Data into Specify 6.pdf)

[2] GBIF (2010). Darwin Core Archives – How-to Guide, version 1, released on 1 March 2011, (contributed by Remsen D, Braak, K, Döring M, Robertson, T), Copenhagen: Global Biodiversity Information Facility, 21 pp, accessible online at: [*http://links.gbif.org/gbif\_dwc-a\_how\_to\_guide\_en\_v1*](http://links.gbif.org/gbif_dwc-a_how_to_guide_en_v1)

## Appendix A

The zipped file-archive should include the following 7 files (each corresponding to a table in Specify with the same name):

- collectionobject.csv
- preparation.csv\*
- collectingevent.csv
- locality.csv
- collector.csv
- determination.csv
- agent.csv

\*) some data contained in the file preparation.csv will be uploaded to the table “preptype”.

Below is a description of the data to be uploaded. The lists include all fields in the 7 corresponding Specify tables. Columns marked with “U” may contain data provided by the user; columns marked with “G” indicate Specify-fields where data is to be generated by the tool during import. These fields should not be included in the text-files supplied by the user. Columns marked with “R” are where the user establish relationships between records in the text-files, and between text-files and existing tree nodes. Other columns are marked with a dash and will not be considered now. Columns not present in the Specify-tables appear in **boldface**.

### collectionobject.csv

| **Column**                  | **Upload/Generate/Reference** | **Notes**                                       |
|-----------------------------|-------------------------------|-------------------------------------------------|
| **CreatedByAgentGUID**      | R                             | Reference to column GUID in agent.csv           |
| **ModifiedByAgentGUID**     | R                             | Reference to column GUID in agent.csv           |
| **CollectingEventGUID**     | R                             | Reference to column GUID in collectingevent.csv |
| **CatalogerAgentGUID**      | R                             | Reference to column GUID in agent.csv           |
| CollectionObjectID          | G                             |                                                 |
| TimestampCreated            | U                             |                                                 |
| TimestampModified           | U                             |                                                 |
| Version                     | G                             |                                                 |
| CollectionMemberID          | G                             |                                                 |
| AltCatalogNumber            | U                             |                                                 |
| Availability                | U                             |                                                 |
| CatalogNumber               | U                             |                                                 |
| CatalogedDate               | U                             |                                                 |
| CatalogedDatePrecision      | U                             |                                                 |
| CatalogedDateVerbatim       | U                             |                                                 |
| CountAmt                    | U                             |                                                 |
| Deaccessioned               | U                             |                                                 |
| Description                 | U                             |                                                 |
| FieldNumber                 | U                             |                                                 |
| GUID                        | U                             | Use this field for linking csv-files            |
| InventoryDate               | U                             |                                                 |
| Modifier                    | U                             |                                                 |
| Name                        | U                             |                                                 |
| Notifications               | U                             |                                                 |
| Number1                     | U                             |                                                 |
| Number2                     | U                             |                                                 |
| ObjectCondition             | U                             |                                                 |
| ProjectNumber               | U                             |                                                 |
| Remarks                     | U                             |                                                 |
| Restrictions                | U                             |                                                 |
| Text1                       | U                             |                                                 |
| Text2                       | U                             |                                                 |
| TotalValue                  | U                             |                                                 |
| OCR                         | U                             |                                                 |
| Visibility                  | U                             |                                                 |
| YesNo1                      | U                             |                                                 |
| YesNo2                      | U                             |                                                 |
| YesNo3                      | U                             |                                                 |
| YesNo4                      | U                             |                                                 |
| YesNo5                      | U                             |                                                 |
| YesNo6                      | U                             |                                                 |
| AccessionID                 | –                             |                                                 |
| CreatedByAgentID            | G                             |                                                 |
| CollectionObjectAttributeID | –                             |                                                 |
| AppraisalID                 | –                             |                                                 |
| ModifiedByAgentID           | G                             |                                                 |
| CollectionID                | G                             |                                                 |
| ContainerOwnerID            | –                             |                                                 |
| ContainerID                 | –                             |                                                 |
| FieldNotebookPageID         | –                             |                                                 |
| VisibilitySetByID           | –                             |                                                 |
| CollectingEventID           | G                             |                                                 |
| PaleoContextID              | –                             |                                                 |
| CatalogerID                 | G                             |                                                 |
| SGRStatus                   | –                             |                                                 |
| ReservedText                | –                             |                                                 |
| Text3                       | U                             |                                                 |
| Integer1                    | U                             |                                                 |
| Integer2                    | U                             |                                                 |
| ReservedInteger3            | –                             |                                                 |
| ReservedInteger4            | –                             |                                                 |
| ReservedText2               | –                             |                                                 |
| ReservedText3               | –                             |                                                 |



### preparation.csv

| **Column**               | **Upload/Generate/Reference** | **Notes**                                          
|--------------------------|-------------------------------|----------------------------------------------------|
| **CreatedByAgentGUID**   | R                             | Reference to column GUID in agent.csv              |
| **ModifiedByAgentGUID**  | R                             | Reference to column GUID in agent.csv              |
| **PrepType**             | U                             | Content in table preptype, field “Name”            |
| **CollectionObjectGUID** | R                             | Reference to column GUID in collectionobject.csv   |
| **PreparedByAgentGUID**  | R                             | Reference to column GUID in agent.csv              |
| **StorageLevel1**        | R                             | Content in storage tree, level 1 (next below root) |
| **StorageLevel2**        | R                             | Content in storage tree, level 2                   |
| **StorageLevel3**        | R                             | Content in storage tree, level 3                   |
| **StorageLevel4**        | R                             | Content in storage tree, level 4                   |
| **StorageLevel5**        | R                             | Content in storage tree, level 5                   |
| PreparationID            | G                             |                                                    |
| TimestampCreated         | U                             |                                                    |
| TimestampModified        | U                             |                                                    |
| Version                  | G                             |                                                    |
| CollectionMemberID       | G                             |                                                    |
| CountAmt                 | U                             |                                                    |
| Description              | U                             |                                                    |
| Number1                  | U                             |                                                    |
| Number2                  | U                             |                                                    |
| PreparedDate             | U                             |                                                    |
| PreparedDatePrecision    | U                             |                                                    |
| Remarks                  | U                             |                                                    |
| SampleNumber             | U                             |                                                    |
| Status                   | U                             |                                                    |
| StorageLocation          | U                             |                                                    |
| Text1                    | U                             |                                                    |
| Text2                    | U                             |                                                    |
| YesNo1                   | U                             |                                                    |
| YesNo2                   | U                             |                                                    |
| YesNo3                   | U                             |                                                    |
| PreparationAttributeID   | –                             |                                                    |
| CreatedByAgentID         | G                             |                                                    |
| ModifiedByAgentID        | G                             |                                                    |
| PrepTypeID               | G                             |                                                    |
| StorageID                | G                             |                                                    |
| CollectionObjectID       | G                             |                                                    |
| PreparedByID             | G                             |                                                    |
| Integer1                 | U                             |                                                    |
| Integer2                 | U                             |                                                    |
| ReservedInteger3         | –                             |                                                    |
| ReservedInteger4         | –                             |                                                    |


### collectingevent.csv

| **Column**                 | **Upload/Generate/Reference** | **Notes**                                |
|----------------------------|-------------------------------|------------------------------------------|
| **LocalityGUID**           | R                             | Reference to column GUID in locality.csv |
| **CreatedByAgentGUID**     | R                             | Reference to column GUID in agent.csv    |
| **ModifiedByAgentGUID**    | R                             | Reference to column GUID in agent.csv    |
| CollectingEventID          | G                             |                                          |
| TimestampCreated           | U                             |                                          |
| TimestampModified          | U                             |                                          |
| Version                    | G                             |                                          |
| EndDate                    | U                             |                                          |
| EndDatePrecision           | U                             |                                          |
| EndDateVerbatim            | U                             |                                          |
| EndTime                    | U                             |                                          |
| Method                     | U                             |                                          |
| Remarks                    | U                             |                                          |
| StartDate                  | U                             |                                          |
| StartDatePrecision         | U                             |                                          |
| StartDateVerbatim          | U                             |                                          |
| StartTime                  | U                             |                                          |
| StationFieldNumber         | U                             |                                          |
| VerbatimDate               | U                             |                                          |
| VerbatimLocality           | U                             |                                          |
| Visibility                 | U                             |                                          |
| LocalityID                 | G                             |                                          |
| VisibilitySetByID          | –                             |                                          |
| CollectingEventAttributeID | –                             |                                          |
| CreatedByAgentID           | G                             |                                          |
| ModifiedByAgentID          | G                             |                                          |
| CollectingTripID           | –                             |                                          |
| DisciplineID               | G                             |                                          |
| SGRStatus                  | –                             |                                          |
| GUID                       | U                             | Use this field for linking csv-files     |
| Integer1                   | U                             |                                          |
| Integer2                   | U                             |                                          |
| ReservedInteger3           | –                             |                                          |
| ReservedInteger4           | –                             |                                          |
| ReservedText1              | U                             |                                          |
| ReservedText2              | U                             |                                          |
| Text1                      | U                             |                                          |
| Text2                      | U                             |                                          |
| PaleoContextID             | –                             |                                          |


### locality.csv

| **Column**            | **Upload/Generate/Reference** | **Notes**                                            |
|-----------------------|-------------------------------|------------------------------------------------------|
| **GeographyLevel1**   | R                             | Content in geography tree, level 1 (next below root) |
| **GeographyLevel2**   | R                             | Content in geography tree, level 2                   |
| **GeographyLevel3**   | R                             | Content in geography tree, level 3                   |
| **GeographyLevel4**   | R                             | Content in geography tree, level 4                   |
| LocalityID            | G                             |                                                      |
| TimestampCreated      | U                             |                                                      |
| TimestampModified     | U                             |                                                      |
| Version               | G                             |                                                      |
| Datum                 | U                             |                                                      |
| ElevationAccuracy     | U                             |                                                      |
| ElevationMethod       | U                             |                                                      |
| GML                   | U                             |                                                      |
| GUID                  | U                             | Use this field for linking csv-files                 |
| Lat1Text              | U                             |                                                      |
| Lat2Text              | U                             |                                                      |
| LatLongAccuracy       | U                             |                                                      |
| LatLongMethod         | U                             |                                                      |
| LatLongType           | U                             |                                                      |
| Latitude1             | U                             |                                                      |
| Latitude2             | U                             |                                                      |
| LocalityName          | U                             |                                                      |
| Long1Text             | U                             |                                                      |
| Long2Text             | U                             |                                                      |
| Longitude1            | U                             |                                                      |
| Longitude2            | U                             |                                                      |
| MaxElevation          | U                             |                                                      |
| MinElevation          | U                             |                                                      |
| NamedPlace            | U                             |                                                      |
| OriginalElevationUnit | U                             |                                                      |
| OriginalLatLongUnit   | U                             |                                                      |
| RelationToNamedPlace  | U                             |                                                      |
| Remarks               | U                             |                                                      |
| ShortName             | U                             |                                                      |
| SrcLatLongUnit        | U                             |                                                      |
| Text1                 | U                             |                                                      |
| Text2                 | U                             |                                                      |
| VerbatimElevation     | U                             |                                                      |
| Visibility            | U                             |                                                      |
| ModifiedByAgentID     | G                             |                                                      |
| GeographyID           | G                             |                                                      |
| DisciplineID          | G                             |                                                      |
| CreatedByAgentID      | G                             |                                                      |
| VisibilitySetByID     | –                             |                                                      |
| SGRStatus             | G                             |                                                      |
| Text3                 | U                             |                                                      |
| Text4                 | U                             |                                                      |
| Text5                 | U                             |                                                      |
| VerbatimLatitude      | U                             |                                                      |
| VerbatimLongitude     | U                             |                                                      |
| PaleoContextID        | –                             |                                                      |


### collector.csv

| **Column**              | **Upload/Generate/Reference** | **Notes**                                       |
|-------------------------|-------------------------------|-------------------------------------------------|
| **CollectorAgentGUID**  | R                             | Reference to column GUID in agent.csv           |
| **CollectingEventGUID** | R                             | Reference to column GUID in collectingevent.csv |
| **ModifiedByAgentGUID** | R                             | Reference to column GUID in agent.csv           |
| **CreatedByAgentGUID**  | R                             | Reference to column GUID in agent.csv           |
| CollectorID             | G                             |                                                 |
| TimestampCreated        | U                             |                                                 |
| TimestampModified       | U                             |                                                 |
| Version                 | G                             |                                                 |
| IsPrimary               | U                             |                                                 |
| OrderNumber             | U                             |                                                 |
| Remarks                 | U                             |                                                 |
| AgentID                 | G                             |                                                 |
| CollectingEventID       | G                             |                                                 |
| ModifiedByAgentID       | G                             |                                                 |
| CreatedByAgentID        | G                             |                                                 |
| DivisionID              | G                             |                                                 |


### determination.csv

| **Column**               | **Upload/Generate/Reference** | **Notes**                                        |
|--------------------------|-------------------------------|--------------------------------------------------|
| **CollectionObjectGUID** | R                             | Reference to column GUID in collectionobject.csv |
| **CreatedByAgentGUID**   | R                             | Reference to column GUID in agent.csv            |
| **TaxonGUID**            | R                             | Reference to GUID for node in taxon tree         |
| **PreferredTaxonGUID**   | R                             | Reference to GUID for node in taxon tree         |
| **ModifiedByAgentGUID**  | R                             | Reference to column GUID in agent.csv            |
| **DeterminerAgentGUID**  | R                             | Reference to column GUID in agent.csv            |
| DeterminationID          | G                             |                                                  |
| TimestampCreated         | U                             |                                                  |
| TimestampModified        | U                             |                                                  |
| Version                  | G                             |                                                  |
| CollectionMemberID       | G                             |                                                  |
| Addendum                 | U                             |                                                  |
| AlternateName            | U                             |                                                  |
| Confidence               | U                             |                                                  |
| DeterminedDate           | U                             |                                                  |
| DeterminedDatePrecision  | U                             |                                                  |
| FeatureOrBasis           | U                             |                                                  |
| IsCurrent                | U                             |                                                  |
| Method                   | U                             |                                                  |
| NameUsage                | U                             |                                                  |
| Number1                  | U                             |                                                  |
| Number2                  | U                             |                                                  |
| Qualifier                | U                             |                                                  |
| VarQualifer              | U                             |                                                  |
| SubSpQualifier           | U                             |                                                  |
| VarQualifier             | U                             |                                                  |
| Remarks                  | U                             |                                                  |
| Text1                    | U                             |                                                  |
| Text2                    | U                             |                                                  |
| TypeStatusName           | U                             |                                                  |
| YesNo1                   | U                             |                                                  |
| YesNo2                   | U                             |                                                  |
| GUID                     | U                             |                                                  |
| CollectionObjectID       | G                             |                                                  |
| CreatedByAgentID         | G                             |                                                  |
| TaxonID                  | G                             |                                                  |
| PreferredTaxonID         | G                             |                                                  |
| ModifiedByAgentID        | G                             |                                                  |
| DeterminerID             | G                             |                                                  |


### agent.csv

| **Column**              | **Upload/Generate/Reference** | **Notes**                             |
|-------------------------|-------------------------------|---------------------------------------|
| **CreatedByAgentGUID**  | R                             | Reference to column GUID in agent.csv |
| **ModifiedByAgentGUID** | R                             | Reference to column GUID in agent.csv |
| AgentID                 | G                             |                                       |
| TimestampCreated        | U                             |                                       |
| TimestampModified       | U                             |                                       |
| Version                 | G                             |                                       |
| Abbreviation            | U                             |                                       |
| AgentType               | U                             |                                       |
| DateOfBirth             | U                             |                                       |
| DateOfBirthPrecision    | U                             |                                       |
| DateOfDeath             | U                             |                                       |
| DateOfDeathPrecision    | U                             |                                       |
| Email                   | U                             |                                       |
| FirstName               | U                             |                                       |
| GUID                    | U                             | Use this field for linking csv-files  |
| Initials                | U                             |                                       |
| Interests               | U                             |                                       |
| JobTitle                | U                             |                                       |
| LastName                | U                             |                                       |
| MiddleInitial           | U                             |                                       |
| Remarks                 | U                             |                                       |
| Title                   | U                             |                                       |
| DateType                | U                             |                                       |
| URL                     | U                             |                                       |
| DivisionID              | G                             |                                       |
| InstitutionCCID         | –                             |                                       |
| InstitutionTCID         | –                             |                                       |
| SpecifyUserID           | –                             |                                       |
| ParentOrganizationID    | –                             |                                       |
| CreatedByAgentID        | G                             |                                       |
| CollectionTCID          | –                             |                                       |
| ModifiedByAgentID       | G                             |                                       |
| CollectionCCID          | –                             |                                       |
| Suffix                  | U                             |                                       |



