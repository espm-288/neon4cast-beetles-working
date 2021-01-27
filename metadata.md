
Helper functions to construct EML, primary to automate the building of
attributeList from the data.frame.

``` r
source("R/meta.R")
```

Specify core metadata as list objects:

``` r
carl <- "https://orcid.org/0000-0002-1642-628X"
authors <- list(list(individualName = list(givenName = "Carl", surName = "Boettiger"), 
                id = carl),
                list(individualName = list(givenName = "Anna",  surName ="Spiers")),
                list(individualName = list(givenName = "Tad", surName = "Dallas")),
                list(individualName = list(givenName = "Brett", surName = "Melbourne"))
                )

tables <- list(
  list(file = "products/richness.csv", 
       description = "Measured carabid beetle species richness
          by siteID and collectDate across all NEON sites operating pitfall traps"),
  list(file = "products/richness_forecast.csv", 
       description = "Forecast of carabid beetle species richness
          by siteID and collectDate across all NEON sites operating pitfall traps")
)
```

Pre-compute geographic coverage from NEON EML, and also generate
taxonomic coverage. Parsing thousands of NEON EML is inefficient, we
really need only one EML per site, so would be better if we restricted
the file list to a YYYY-MM when all sites were sampling. To streamline
things, we store the coverage data instead. (Alternately we could have
constructed the geographicCoverage from the NEON API locations
endpoint). Note that NEON EML lists each NEON site as effectively a
point location, despite EML being defined as a bounding box.

``` r
library(dplyr)
library(jsonlite)
meta <- neonstore::neon_index(ext="xml", product = "DP1.10022.001")
all <- lapply(meta$path, emld::as_emld)
geo <- lapply(all, function(x) x$dataset$coverage$geographicCoverage)
geo %>% toJSON() %>% fromJSON() %>% distinct() %>% write_json("meta/bet_geo.json", auto_unbox=TRUE)
```

``` r
library(dplyr)
library(jsonlite)
library(EML)
taxonomicCoverage <- EML::set_taxonomicCoverage(
  data.frame(Kingdom = "Animalia", Phylum="Insecta", Class = "Coleoptera", Family="Carabidae"))
taxonomicCoverage %>% jsonlite::write_json("meta/carabid_taxa.json")
```

Load the stored taxonomic and geographic coverage, compute current
temporal coverage:

``` r
richness <- vroom::vroom("products/richness.csv")
```

    ## Rows: 2,188
    ## Columns: 5
    ## Delimiter: ","
    ## chr  [2]: siteID, month
    ## dbl  [2]: year, n
    ## date [1]: collectDate
    ## 
    ## Use `spec()` to retrieve the guessed column specification
    ## Pass a specification to the `col_types` argument to quiet this message

``` r
startDate <- min(richness$collectDate)
endDate <- max(richness$collectDate) # Or is this the forecast horizon
temporalCoverage <- 
              list(rangeOfDates =
                list(beginDate = list(calendarDate = startDate),
                     endDate = list(calendarDate = endDate)))
                     
geographicCoverage <- jsonlite::read_json("meta/bet_geo.json")
taxonomicCoverage <-  jsonlite::read_json("meta/carabid_taxa.json")

coverage <- list(geographicCoverage = geographicCoverage,
                 temporalCoverage = temporalCoverage,
                 taxonomicCoverage = taxonomicCoverage)
```

Now generate EML. Note that `attributesList` will be computed from the
`vroom::spec` of the data files, which handles data types reasonably but
is fast and loose on units. Fortunately `dimensionless` is okay for the
count data included here.

``` r
meta <- build_eml(title = "NEON Carabid Species Richness forecast", 
          abstract = "Simple forecast of Carabid beetle species richness at
                     each month at each NEON site for 2019, based on historical averages.", 
          creators = authors, 
          contact_orcid = carl,
          coverage = coverage,
          tables = tables)
```

    ## Rows: 1
    ## Columns: 5
    ## Delimiter: ","
    ## chr  [2]: siteID, month
    ## dbl  [2]: year, n
    ## date [1]: collectDate
    ## 
    ## Use `spec()` to retrieve the guessed column specification
    ## Pass a specification to the `col_types` argument to quiet this message

    ## Rows: 1
    ## Columns: 5
    ## Delimiter: ","
    ## chr [2]: month, siteID
    ## dbl [3]: mean, sd, year
    ## 
    ## Use `spec()` to retrieve the guessed column specification
    ## Pass a specification to the `col_types` argument to quiet this message

``` r
emld::as_xml(meta, "meta/eml.xml")
```

    ## NULL

``` r
emld::eml_validate("meta/eml.xml")
```

    ## [1] TRUE
    ## attr(,"errors")
    ## character(0)

## Publish the product to DataONE:

Helper functions which wrap around the standard `dataone` R package
functions. Main difference is to default to content-based identifiers
(hash URIs) for each object, and to set sha256 as the checksum used for
objects. Also adds relationships and provenance.

To actually upload to the production KNB server, you will need to have a
`dataone_token` defined in `options()`. For this to run in test mode,
you will need to have a `dataone_test_token` defined in `options()`.
Having a test token available forces use of the dataone testing server,
even if a production token is also available.

``` r
source("R/dataone.R")
```

    ## 
    ## Attaching package: 'mime'

    ## The following object is masked from 'package:vroom':
    ## 
    ##     guess_type

``` r
publish_dataone(in_file="products/richness.csv", 
                out_file="products/richness_forecast.csv", 
                code="workflow.Rmd", 
                meta="meta/eml.xml",
                orcid=carl)
```
