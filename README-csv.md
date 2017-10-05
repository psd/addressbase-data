# Load AddressBase CSV files into PostgreSQL

Download the single zip file of non-5km CSV files from OrdnanceSurvey.  Unzip
them into `cache/AddressBase/csv/`

Each file contains records of different types, with different numbers of
columns.  The type is given by the first column.  Separate them by type, and
combine the types into single files, using `awk`.  It will take minutes, not
hours.

```sh
mkdir by_type
awk -F, '{print > "by_type/"$1".csv"}' data/*.csv
```

Create tables in Postgres, and import the data into them.  This should take only
a few minutes, even for the largest tables.

```sh
# Some tables commented-out because you probably don't need them.

# To decide which ones you need, refer to sql/create-tables.pgsql for the column
# headers.

createdb addressbase

psql -d addressbase -a -f sql/drop-tables.pgsql
psql -d addressbase -a -f sql/create-tables.pgsql

psql -d addressbase -c "\copy basiclandpropertyunit FROM 'cache/AddressBase/csv/by_type/21.csv' delimiter ',' csv;"
# psql -d addressbase -c "\copy classification FROM 'cache/AddressBase/csv/by_type/32.csv' delimiter ',' csv;"
psql -d addressbase -c "\copy deliverypointaddress FROM 'cache/AddressBase/csv/by_type/28.csv' delimiter ',' csv;"
psql -d addressbase -c "\copy landpropertyidentifier FROM 'cache/AddressBase/csv/by_type/24.csv' delimiter ',' csv;"
# psql -d addressbase -c "\copy organisation FROM 'cache/AddressBase/csv/by_type/31.csv' delimiter ',' csv;"
# psql -d addressbase -c "\copy applicationcrossreference FROM 'cache/AddressBase/csv/by_type/23.csv' delimiter ',' csv;"
# psql -d addressbase -c "\copy street FROM 'cache/AddressBase/csv/by_type/11.csv' delimiter ',' csv;"
psql -d addressbase -c "\copy streetdescriptiveidentifier FROM 'cache/AddressBase/csv/by_type/15.csv' delimiter ',' csv;"
```

```pgsql
CREATE INDEX ON deliverypointaddress (organisation_name);
CREATE INDEX ON basiclandpropertyunit (uprn);
CREATE INDEX ON landpropertyidentifier (uprn, usrn);
CREATE INDEX ON streetdescriptiveidentifier (usrn);
```

Example search

```pgsql
SELECT
  dpa.organisation_name,
  dpa.department_name,
  dpa.postcode,
  blpu.uprn,
  blpu.latitude,
  blpu.longitude,
  sdi.usrn,
  sdi.street_description,
  sdi.locality,
  sdi.town_name
FROM
  deliverypointaddress AS dpa
LEFT JOIN basiclandpropertyunit AS blpu ON blpu.uprn = dpa.uprn
LEFT JOIN landpropertyidentifier AS lpi ON lpi.uprn = dpa.uprn
  AND lpi.language = 'ENG'
  AND lpi.logical_status = 1 -- 1=Approved
                             -- 3=Alternative
                             -- 6=provisional
                             -- 8=historical
  AND lpi.end_date is null
LEFT JOIN streetdescriptiveidentifier AS sdi ON sdi.usrn = lpi.usrn
  AND sdi.language = 'ENG'
  AND sdi.end_date is null
WHERE dpa.organisation_name LIKE '%JOB%CENTRE%'
  AND lpi.logical_status = 1
;
```

#### Geospatial

I don't know whether it's worth making this into a spatial database or not, but
at any rate this is how to do it.

```pgsql
CREATE EXTENSION postgis;
SELECT AddGeometryColumn ('basiclandpropertyunit', 'geom', 4258, 'POINT', 2);
UPDATE basiclandpropertyunit SET geom = ST_SetSRID(ST_MakePoint(longitude, latitude), 4258);
```

#### Elasticsearch

This doesn't work yet.

A good integration with ElasticSearch seems to be
[zombodb](https://www.zombodb.com/)
([tutorial](https://github.com/zombodb/zombodb/blob/master/TUTORIAL.md)), but it
doesn't support PostgreSQL v.9.6 and will wait for v.10.

[Download](https://www.zombodb.com/releases/) the latest version of both the
ElasticSearch `.zip` and PostgreSQL `.deb` extensions.
