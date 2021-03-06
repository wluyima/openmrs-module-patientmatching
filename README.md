# Patient Matching Module

Patient matching is a specific application, in which we try to identify records that belong to the same patient among different data sources. These sources can range from patient data collected at different hospitals to external information from governmental institutions, such as death master file etc.

#### Table of Contents
1. [What The Module Does](#)
2. [Patient Matching Strategy](#)
3. [Module Configuration](#)
4. [Configuration File](#)
5. [String Comparators (Algorithms)](#)
6. [Scheduling Patient Matching Tasks](#)
7. [Patient Matching Attribute](#)

#### What The Module Does

This OpenMRS module wraps around the patientmatching jar to facilitate creating the matching file setup and merging patients that are found to be duplicates.

For full documentation, please visit: https://wiki.openmrs.org/display/docs/Patient+Matching+Module

#### Patient Matching Strategy

**Probabilistic**

This is the default strategy that requires training the patient matching engine how to match records, it keeps
details about matched and unmatched fields whenever any 2 records are matched and uses it to determine scores
for future comparisons. For example, if a certain property matches 90% of the times for true matched records
then it ends up carrying more weight over fields that have lower true match percentages. This has a huge
implication, it means given the same records to compare it can produce different scores potentially matching or not
matching them for different runs depending on the current m(true) and u(false) values of the properties
being compared but overtime it should normalize.

**Deterministic**

Because of the somewhat unpredictable nature of probabilistic strategy, some implementation find it undesirable
especially if they have more straight forward and simpler patient matching needs to address. This strategy
guarantees the same results given the same configuration and records to match, requires no training of the
matching engine and is much simpler to configure.

The matching strategy can be configured via the patientmatching.strategy global property.

#### Module Configuration

**patientmatching.strategy**

The patient matching strategy to use, allowed values are probabilistic and deterministic, defaults to
probabilistic. With deterministic strategy m and u values are ignored therefore any 2 records are considered to
match if all their demographics match based on their respective algorithms.

**patientmatching.configDirectory**

The location of the patient matching configuration folder, can be absolute or relative path, defaults to
patientmatching in the OpenMRS application data directory.

**patientmatching.linkConfigFile**

The location of the patient matching configuration file, can be absolute or relative path, defaults to
patientmatching/link_config.xml in the OpenMRS application data directory.

**patientmatching.serializationDirectory**

The location of the serialization folder to be used when matching patients, can be absolute or relative path,
defaults to patientmatching/serial in the OpenMRS application data directory.
  
**patientmatching.unknownPatientAttributeTypeUuid**

The uuid of the person attribute type that is used to mark a patient as unknown, this assumes the attribute type
 takes strings values where true is the value for the unknown patient.

Please refer to the OpenMRS settings page in the application for a complete list of all global properties for the module.

#### Configuration File
The default name for the configuration file is "link_config.xml" in the patietmatching directory in the OpenMRS application data directory.
The JDBC driver needs to be in the classpath when the program is run if the link table is in a non Postgres or MySQL directory.

The description of the elements and attributes of the xml configuration file is:

**Session** the root element

**datasource** a source of Record objects and below are the attributes,

- **name** for file sources, give the path, for data bases, gives the table name
- **type**  type of datasource: CharDelimFile, DataBase, Vector
- **access** how to access the datasource, optional when running as an OpenMRS module.  For a character delimted file, it is the delimiter.  For a database, it is a String holding connection information
- **id** a numeric unique identifier for the data source

**column** one column of fields in the datasource and below are the attributes,
- **column_id** name of the column.  For a character delimited file, it is an index.  For a database table, it is the column name
- **include_position** if column is a part of the analysis, what order it is.  Zero indexed
- **label** the name used by the linkage program and that appears in the runsection.  It should be the demographics that appear in the matching person attribute
- **type** either is string or numeric and used in sorting and comparisons
    
**run** a set of link options to use with the datasources and below are the attributes,
- **estimate** Whether to use EM to modify values
- **name** a label for this configuration

**row** the options for a field in the Record and below are the attributes,
- **name** the name of the field, must match the label in the Datasource element

**BlockOrder** if the field is a blocking field, then uniquely number this starting with 1

**BlckChars** the number of characters to block on if the field is a blocking field

**Include** indicates if the field will be compared between records

**TAgreement** the true agreement value

**NonAgreement** the non agreement value

**Threshold** the double value above which 2 field values are considered a match

**ScaleWeight** true for enabling weight scaling, null for disabling and below are the attributes,
- **lookup** Determines the tokens that will be loaded to the lookup table. Possible values are: TopN, TopNPercent, AboveN, BelowN, BottomNPercent, BottomN \
- **N** Defines the size of the lookup table, must be a decimal number, use a number between 0.0 and 1.0 for percentages \
- **buffer** Number of records that will be stored in memory during analysis (no need to exceed the number of unique tokens)

**Algorithm** the comparator to use for this field.  Options are Exact Match, LEV, LCS, and JWC

**SetID** - If the field is set as transposed,then the setId will be a number(the value of the set).if the field is not transposed by default it will be 0

A excerpt of a valid configuration file for the probabilistic strategy is (see below for that the deterministic):
    
    <?xml version="1.0" encoding="UTF-8" ?>
    <Session>
        <datasource name="link_test" type="DataBase" id="3">
            <column include_position="0" column_id="mrn" label="mrn" type="string"/>
            <column include_position="1" column_id="ln" label="ln" type="string"/>
            . . .
            <column include_position="17" column_id="openmrs_id" label="openmrs_id" type="string"/>
        </datasource>
        <analysis type="scaleweight">
        <init>DBCdriver,databaseURL,user,passwd</init>
        </analysis>
        <run estimate="true" name="conversion">
            <row name="yb">
                <BlockOrder>1</BlockOrder>
                <BlckChars>40</BlckChars>
                <Include>false</Include>
                <TAgreement>0.9</TAgreement>
                <NonAgreement>0.1</NonAgreement>
                <ScaleWeight lookup="TopN" N="100.0" buffer="500">true</ScaleWeight>
                <Algorithm>Exact Match</Algorithm>
        <SetID>0</SerID>
            </row>
            . . .
            <row name="zip">
                <BlockOrder>null</BlockOrder>
                <BlckChars>40</BlckChars>
                <Include>true</Include>
                <TAgreement>0.9</TAgreement>
                <NonAgreement>0.1</NonAgreement>
                <ScaleWeight>null</ScaleWeight>
                <Algorithm>Exact Match</Algorithm>
            <SetID>0</SerID></row>
        </run>
    </Session>

Example configuration file for deterministic strategy:

    <?xml version="1.0" encoding="UTF-8" standalone="no"?>
    <Session>
        <datasource header="false" id="1" name="patient" type="DataBase">
            <column column_id="patient_id" include_position="0" label="org.openmrs.Patient.patientId" type="number" />
            <column column_id="gender" include_position="1" label="org.openmrs.Patient.gender" type="string" />
            <column column_id="birthdate" include_position="2" label="org.openmrs.Patient.birthdate" type="string" />
            <column column_id="family_name" include_position="3" label="org.openmrs.PersonName.familyName" type="string" />
            <column column_id="given_name" include_position="4" label="org.openmrs.PersonName.givenName" type="string" />
        </datasource>
        <run estimate="true" name="default-config">
            <row name="org.openmrs.Patient.patientId">
                <BlockOrder>null</BlockOrder>
                <BlckChars>40</BlckChars>
                <Include>false</Include>
            </row>
            <row name="org.openmrs.Patient.gender">
                <BlockOrder>1</BlockOrder>
                <BlckChars>40</BlckChars>
                <Include>true</Include>
            </row>
            <row name="org.openmrs.Patient.birthdate">
                <BlockOrder>2</BlockOrder>
                <BlckChars>40</BlckChars>
                <Include>true</Include>
            </row>
            <row name="org.openmrs.PersonName.familyName">
                <BlockOrder>null</BlockOrder>
                <BlckChars>40</BlckChars>
                <Include>true</Include>
                <Threshold>0.75</Threshold>
            </row>
            <row name="org.openmrs.PersonName.givenName">
                <BlockOrder>null</BlockOrder>
                <BlckChars>40</BlckChars>
                <Include>true</Include>
                <Threshold>0.75</Threshold>
            </row>
        </run>
    </Session>

#### String Comparators (Algorithms)
The string comparators(algorithms) that can be used are:

**Exact Match** Comparison for the whole string, similarity is either 0 or 1

**[Levenshtein](https://en.wikipedia.org/wiki/Levenshtein_distance)** (LEV) Levenshtein edit distance / longest string length

**[Longest Common Substring](https://en.wikipedia.org/wiki/Longest_common_subsequence_problem)** (LCS) Regenstrief algorithm, converts to case insensitive strings in comparison

**[Jaro Winkler](https://en.wikipedia.org/wiki/Jaro%E2%80%93Winkler_distance)** (JWC) A string metric measuring an edit distance between two sequences

**[Dice Coefficient](https://en.wikipedia.org/wiki/S%C3%B8rensen%E2%80%93Dice_coefficient)** (DICE) 

The implementations for Levenshtein and Jaro Winkler comparators come from the [SimMetrics library](https://github.com/Simmetrics/simmetrics).  The default threshold for Jaro Winkler
and Dice Coefficient is 0.8, Longest Common Substring is 0.85 and for Levenshtein is 0.7. But you can overide these values in the configuration file for each field.

#### Scheduling Patient Matching Tasks
You can schedule tasks to run a specified patient matching strategy e.g. daily at midnight. The generated reports can be found at
the module's **Create Report** page. Below are the steps to create a scheduler task:
- Navigate to the legacy UI admin page, for RA users it is _Home -> System Administration -> Advanced Administration_
- Under the **Patient Matching Module** section, click on **Manage Schedule**
- Click **Create Schedule**
- Enter a unique name, be sure to check exactly 1 strategy, set the start time pattern, schedule date and repeat interval
- Click the Save button

**Note:** You can always edit the scheduler task from the main **Manage Scheduler** page which provides more configuration options. 

#### Patient Matching attribute
The module prefers to use a special matching PersonAttributeType of "Other Matching Information".  This is a list of demographic-value pairs in the form of "<demographic1>:<value1>;<demographic2>:<value2>; . . . ".  If a demographic has no value, then it can either not be present in the string of the value can be an empty string.

If there is no person attribute of that type, then the module will try to get some basic information as best it can.  Currently, this is very minimal and would not make good matches.

