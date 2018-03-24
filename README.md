postGIStools
============

[![Travis-CI Build Status](https://travis-ci.org/SESYNC-ci/postGIStools.svg?branch=master)](https://travis-ci.org/SESYNC-ci/postGIStools)

**Note**: This package is deprecated. For new projects, we recommend using the [sf](https://github.com/r-spatial/sf) package to interface with geodatabases.

The postGIStools package extends the standard R / PostgreSQL interface (as implemented in RPostgreSQL) to provide support for two popular PostgreSQL extensions: *PostGIS* (spatial data) and *hstore* (key/value pairs).

How to install
--------------

Install the package from this GitHub repository using the following code:

``` r
install.packages("devtools")  # if necessary
devtools::install_github("SESYNC-ci/postGIStools")
```

Reading PostGIS and hstore data into R
--------------------------------------

We demonstrate the postGIStools functions using a test database hosted on Heroku. It contains a single table *country* with the following fields:

<table style="width:82%;">
<colgroup>
<col width="16%" />
<col width="12%" />
<col width="52%" />
</colgroup>
<thead>
<tr class="header">
<th align="left">name</th>
<th align="left">type</th>
<th align="left">comments</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="left">name</td>
<td align="left">text</td>
<td align="left">country name</td>
</tr>
<tr class="even">
<td align="left">iso2</td>
<td align="left">text</td>
<td align="left">ISO two-letter code (primary key)</td>
</tr>
<tr class="odd">
<td align="left">capital</td>
<td align="left">text</td>
<td align="left"></td>
</tr>
<tr class="even">
<td align="left">population</td>
<td align="left">integer</td>
<td align="left"></td>
</tr>
<tr class="odd">
<td align="left">translations</td>
<td align="left">hstore</td>
<td align="left">key/value pairs where key is language (e.g. &quot;es&quot;, &quot;fr&quot;)</td>
</tr>
<tr class="even">
<td align="left">geom</td>
<td align="left">geometry</td>
<td align="left">country polygons</td>
</tr>
</tbody>
</table>

The data originates from the [REST Countries API](https://restcountries.eu/), whereas the country geometries are from the *wrld\_simpl* map included in maptools R package.

To read data from PostgreSQL into R, postGIStools provides the `get_postgis_query` function. Like the `dbGetQuery` function in PostgreSQL, it requires a connection object and a SQL statement, which in this case must be a SELECT statement. In addition, the user may identify a geometry and/or hstore field by name.

``` r
library(RPostgreSQL)
library(postGIStools)

con <- dbConnect(PostgreSQL(), dbname = "d2u06to89nuqei", user = "mzcwtmyzmgalae",
                 host = "ec2-107-22-246-250.compute-1.amazonaws.com",
                 password = "UTv2BuwJUPuruhDqJthcngyyvO")

countries <- get_postgis_query(con, "SELECT * FROM country 
                               WHERE population > 1000000",
                               geom_name = "geom", hstore_name = "translations")

class(countries)
## [1] "SpatialPolygonsDataFrame"
## attr(,"package")
## [1] "sp"
```

When a geometry column is specified, the query output is a spatial data frame type from the sp package. The hstore column is converted to a list-column in R, where each "cell" is a named list.

``` r
str(countries@data[1:2,])
## 'data.frame':    2 obs. of  5 variables:
##  $ name        : chr  "Afghanistan" "Albania"
##  $ iso2        : chr  "AF" "AL"
##  $ capital     : chr  "Kabul" "Tirana"
##  $ population  : int  26023100 2893005
##  $ translations:List of 2
##   ..$ :List of 5
##   .. ..$ de: chr "Afghanistan"
##   .. ..$ es: chr "Afganistán"
##   .. ..$ fr: chr "Afghanistan"
##   .. ..$ it: chr "Afghanistan"
##   .. ..$ ja: chr "アフガニスタン"
##   ..$ :List of 5
##   .. ..$ de: chr "Albanien"
##   .. ..$ es: chr "Albania"
##   .. ..$ fr: chr "Albanie"
##   .. ..$ it: chr "Albania"
##   .. ..$ ja: chr "アルバニア"
```

The query statement can include the output of PostGIS spatial functions. In that case, it is simpler to set an alias for the output column and pass that alias as the `geom_name`. For example, the following query returns a *SpatialPointsDataFrame* of the country centroids.

```r
centroids <- get_postgis_query(con, 
                        "SELECT name, ST_Centroid(geom) centr FROM country",
                        geom_name = "centr")
head(centroids)
##              coordinates           name
## 1   (66.00111, 33.85878)    Afghanistan
## 2   (19.97356, 60.20129)  Åland Islands
## 3   (20.07437, 41.13255)        Albania
## 4   (2.627813, 28.17211)        Algeria
## 5 (-169.8823, -14.25412) American Samoa
## 6   (1.650388, 42.56045)        Andorra
```


Working with hstore columns
---------------------------

To interact with hstore columns imported into R, postGIStools defines the `%->%` operator, which is analogous to `->` in PostgreSQL. Specifically, `hstore %->% "key"` extracts the value in each cell of the hstore corresponding to the given key, or `NA` when the key is absent from a given cell.

``` r
head(countries$translations %->% "es")
## [1] "Afganistán" "Albania"    "Argelia"    "Angola"     "Argentina" 
## [6] "Armenia"
```

The operator is also compatible with single bracket subsetting of the hstore.

``` r
countries$translations[5:7] %->% "fr"
## [1] "Argentine" "Arménie"   "Australie"
```

The assignment version of `%->%` operates similarly, with the option of deleting keys by assigning them to `NULL`.

``` r
countries$translations[2] %->% "nl" <- "Albanië"
countries$translations[3] %->% "fr" <- NULL
countries$translations[2:3]
## [[1]]
## [[1]]$de
## [1] "Albanien"
## 
## [[1]]$es
## [1] "Albania"
## 
## [[1]]$fr
## [1] "Albanie"
## 
## [[1]]$it
## [1] "Albania"
## 
## [[1]]$ja
## [1] "アルバニア"
## 
## [[1]]$nl
## [1] "Albanië"
## 
## 
## [[2]]
## [[2]]$de
## [1] "Algerien"
## 
## [[2]]$es
## [1] "Argelia"
## 
## [[2]]$it
## [1] "Algeria"
## 
## [[2]]$ja
## [1] "アルジェリア"
```

The `new_hstore` function creates a blank hstore of a given length, which is just an empty list of lists. It is most useful when assigned to a data frame column (e.g. `df$hs <- new_hstore(3)`) that can be then populated with `%->%` and written back to a PostgreSQL database.

Inserting and updating PostgreSQL tables from R
-----------------------------------------------

The two write methods `postgis_insert` and `postgis_update` wrap around their namesake SQL commands, while also converting R spatial objects and list-columns back into the geometry and hstore data types, respectively.

To demonstrate these functions, we create a new temporary table in the database.

``` r
dbSendQuery(con, paste("CREATE TEMP TABLE cty_tmp (name text,", 
                       "iso2 text PRIMARY KEY, capital text,",
                       "translations hstore, geom geometry)"))
```

Calls to `postgis_insert` must specify the connection, data frame and table name. By default, all data frame columns are inserted, but a subset of columns can be specified as `write_cols`. In both cases, the names of inserted columns must have a match in the target table.

``` r
postgis_insert(con, countries[1:10,], "cty_tmp",
               write_cols = c("name", "iso2", "translations"),
               geom_name = "geom", hstore_name = "translations")
## <PostgreSQLResult:(39779,0,4)>

# Reimport to check
cty_tmp <- get_postgis_query(con, paste("SELECT name, iso2, capital,",
                                        "geom, translations FROM cty_tmp"),
                             geom_name = "geom", hstore_name = "translations")
head(cty_tmp@data)
##          name iso2 capital
## 1 Afghanistan   AF    <NA>
## 2     Albania   AL    <NA>
## 3     Algeria   DZ    <NA>
## 4      Angola   AO    <NA>
## 5   Argentina   AR    <NA>
## 6     Armenia   AM    <NA>
##                                                        translations
## 1 Afghanistan, Afganistán, Afghanistan, Afghanistan, アフガニスタン
## 2          Albanien, Albania, Albanie, Albania, アルバニア, Albanië
## 3                          Algerien, Argelia, Algeria, アルジェリア
## 4                          Angola, Angola, Angola, Angola, アンゴラ
## 5        Argentinien, Argentina, Argentine, Argentina, アルゼンチン
## 6                   Armenien, Armenia, Arménie, Armenia, アルメニア
```

We next update the records in *cty\_tmp* to include the *capital* field. The syntax of `postgis_update` is similar to `postgis_insert`, except that we must specify both `id_cols`, the column(s) identifying the records to update, as well as `update_cols`, the column(s) to be updated. (The underlying PostgreSQL operation is of the format *UPDATE... SET ... FROM...*.) Neither the geometry nor the hstore can be used as `id_cols`. Note that since the input data frame `countries[1:10,]` includes spatial and list-column data, we need to specify `geom_name` and `hstore_name`, even if those columns are not needed for the update operation.

``` r
postgis_update(con, countries[1:10,], "cty_tmp", id_cols = "iso2", 
               update_cols = "capital", geom_name = "geom", 
               hstore_name = "translations")
## <PostgreSQLResult:(39779,0,7)>

cty_tmp <- get_postgis_query(con, paste("SELECT name, iso2, capital,",
                                        "geom, translations FROM cty_tmp"),
                             geom_name = "geom", hstore_name = "translations")
head(cty_tmp@data)
##          name iso2      capital
## 1   Australia   AU     Canberra
## 2 Afghanistan   AF        Kabul
## 3     Albania   AL       Tirana
## 4     Algeria   DZ      Algiers
## 5      Angola   AO       Luanda
## 6   Argentina   AR Buenos Aires
##                                                        translations
## 1       Australien, Australia, Australie, Australia, オーストラリア
## 2 Afghanistan, Afganistán, Afghanistan, Afghanistan, アフガニスタン
## 3          Albanien, Albania, Albanie, Albania, アルバニア, Albanië
## 4                          Algerien, Argelia, Algeria, アルジェリア
## 5                          Angola, Angola, Angola, Angola, アンゴラ
## 6        Argentinien, Argentina, Argentine, Argentina, アルゼンチン
```

By default, hstore columns are updated by concatenation: keys present in the input data frame but not the original table are added to the hstore, keys present in both the data frame and table have their associated values updated, but keys absent from the input data frame are *not* deleted from the table. This can be changed by setting `hstore_concat = FALSE`, in which case whole hstore cells are replaced with corresponding ones in the input data frame.

``` r
countries$translations[2] %->% "nl" <- NULL
countries$translations[3] %->% "fr" <- "Algérie"
 
postgis_update(con, countries[1:10,], "cty_tmp", id_cols = "iso2", 
               update_cols = "translations", geom_name = "geom", 
               hstore_name = "translations")
## <PostgreSQLResult:(39779,0,10)>

cty_tmp <- get_postgis_query(con, paste("SELECT name, iso2, capital,",
                                        "geom, translations FROM cty_tmp"),
                             geom_name = "geom", hstore_name = "translations")
cty_tmp@data[cty_tmp$iso2 %in% c("AL", "DZ"), ]
##      name iso2 capital
## 2 Albania   AL  Tirana
## 5 Algeria   DZ Algiers
##                                               translations
## 2 Albanien, Albania, Albanie, Albania, アルバニア, Albanië
## 5        Algerien, Argelia, Algérie, Algeria, アルジェリア

# Key deletion not reflected in database unless hstore_concat = FALSE
postgis_update(con, countries[1:10,], "cty_tmp", id_cols = "iso2", 
               update_cols = "translations", geom_name = "geom", 
               hstore_name = "translations", hstore_concat = FALSE)
## <PostgreSQLResult:(39779,0,13)>

cty_tmp <- get_postgis_query(con, paste("SELECT name, iso2, capital,",
                                        "geom, translations FROM cty_tmp"),
                             geom_name = "geom", hstore_name = "translations")
cty_tmp@data[cty_tmp$iso2 %in% c("AL", "DZ"), ]
##      name iso2 capital                                      translations
## 2 Albania   AL  Tirana   Albanien, Albania, Albanie, Albania, アルバニア
## 5 Algeria   DZ Algiers Algerien, Argelia, Algérie, Algeria, アルジェリア
```

References
----------

The code to import geom fields is based on blog post from Lee Hachadoorian: [Load PostGIS geometries in R without rgdal](http://www.r-bloggers.com/load-postgis-geometries-in-r-without-rgdal/)

Acknowledgements
----------------

Development of this R package was supported by the National Socio-Environmental Synthesis Center (SESYNC) under funding received from the National Science Foundation DBI-1052875.
