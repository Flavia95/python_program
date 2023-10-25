## Overview [insert_data.py](https://gitlab.com/fredmanglis/gnqc_py/-/blob/main/scripts/insert_data.py?ref_type=heads) 

This script is designed to insert either average data or standard error data into a database using input from a file.
These functions collectively provide the necessary functionality to read and manipulate data from the input file and interact with the database to insert data based on the type of data (averages or standard errors) provided.

1. This function takes a heading as input, which is typically a strain alias. It translates strain aliases into canonical names using a dictionary called translations.
```
def translate_alias(heading):
    "Translate strain aliases into canonical names"
    translations = {"B6": "C57BL/6J", "D2": "DBA/2J"}
    return translations.get(heading, heading)
```
2. This function reads the headers (column names) from the input file specified by filepath. It opens the file, reads the first line. It returns the headings as a tuple.

```
def read_file_headings(filepath) -> Tuple[str, ...]:
    "Get the file headings"
    with open_file(filepath) as input_file:
        for line_contents in input_file:
            headings = tuple(
                translate_alias(heading.strip())
                for heading in line_contents.split("\t"))
            break

    return headings

```
3. This function reads the content of the input file specified by filepath. It iterates through the file line by line, skipping the first line, and yields each line's data as a tuple.
   
```
def read_file_contents(filepath):
    "Get the file contents"
    with open_file(filepath) as input_file:
        for line_number, line_contents in enumerate(input_file):
            if line_number == 0:
                continue
            if line_number > 0:
                yield tuple(
                    field.strip() for field in line_contents.split("\t"))
```
4. This function retrieves information for the strains from the database. It takes a database connection (dbconn), a tuple of strain names, and a species ID as input.
It constructs a SQL query to fetch strain information for the provided strain names and species ID from the database. The function returns a dictionary where strain names are keys, and the corresponding strain information is the value.
```
def strains_info(
        dbconn: mdb.Connection, strain_names: Tuple[str, ...],
        speciesid: int) -> dict:
    "Retrieve information for the strains"
    with dbconn.cursor(cursorclass=DictCursor) as cursor:
        query = (
            "SELECT * FROM Strain WHERE Name IN "
            f"({', '.join(['%s']*len(strain_names))}) "
            "AND SpeciesId = %s")
        cursor.execute(query, tuple(strain_names) + (speciesid,))
        return {strain["Name"]: strain for strain in cursor.fetchall()}
```
5. This function reads data values from the file, given the file path, headers, and strain information. It reads data values line by line, creating a dictionary for each line, with keys being column names from the header and values as the corresponding data. The function yields these dictionaries.
```
def read_datavalues(filepath, headings, strain_info):
    "Read data values from file"
    for row in (
            dict(zip(headings, line))
            for line in read_file_contents(filepath)):
        for sname in headings[1:]:
            yield {
                "ProbeSetId": int(row["ProbeSetID"]),
                "StrainId": strain_info[sname]["Id"],
                "DataValue": float(row[sname])
            }
```
6. This function retrieves the last ID from the database, specifically from the ProbeSetData table. It constructs an SQL query to find the maximum ID and returns the result as an integer.

```
def last_data_id(dbconn: mdb.Connection) -> int:
    "Get the last id from the database"
    with dbconn.cursor() as cursor:
        cursor.execute("SELECT MAX(Id) FROM ProbeSetData")
        return int(cursor.fetchone()[0])
```
7. This function checks if the strains mentioned in the input file's header exist in the database. It takes a list of strains from the file's header and a dictionary of strains from the database. If any strains are in the input file that do not exist in the database, it prints an error message and exits the program.

```
def check_strains(headings_strains, db_strains):
    "Check strains in headings exist in database"
    from_db = tuple(db_strains.keys())
    not_in_db = tuple(
        strain for strain in headings_strains if strain not in from_db)
    if len(not_in_db) == 0:
        return True

    str_not_in_db = "', '".join(not_in_db)
    print(
        (f"ERROR: The strain(s) '{str_not_in_db}' w(as|ere) not found in the "
         "database."),
        file=sys.stderr)
    sys.exit(1)

```
8. This function retrieves annotation information from the database. It constructs a SQL query to select annotation information for a specific platform and dataset. The resulting information is organized into dictionaries, where probe names and target IDs are used as keys.
```
def annotationinfo(
        dbconn: mdb.Connection, platformid: int, datasetid: int) -> dict:
    # This is somewhat slow. Look into optimising the behaviour
    def __organise_annotations__(accm, item):
        names_dict = (
            {**accm[0], item["Name"]: item} if bool(item["Name"]) else accm[0])
        targs_dict = (
            {**accm[1], item["TargetId"]: item}
            if bool(item["TargetId"]) else accm[1])
        return (names_dict, targs_dict)
```

The *insert_means* and *insert_se* functions are responsible for inserting average data or standard error data into the database. They use data extracted from the input file, link strain and annotation information from the database, and then execute insertion queries.

### What the data should be?

-  Must be (TSV) files, that contain values tab-separated.
-   This is the header that should works:
```
Id	Name	Name2	SpeciesId	Symbol	Alias
```
- Check line endings, lines should ending uses just line feed ("\n").
-  You shouldn't use data with empty cells.
-   No data cells with special characters are allowed, i.e. iie)
-   For decimal numbers there must be a number to the left side of the dot (e.g. 0.4444)
