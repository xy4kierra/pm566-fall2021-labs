lab07-pm566
================
Xiaoyu Zhu
10/8/2021

## packages

``` r
library(tidyverse)
```

    ## ── Attaching packages ─────────────────────────────────────── tidyverse 1.3.1 ──

    ## ✓ ggplot2 3.3.5     ✓ purrr   0.3.4
    ## ✓ tibble  3.1.5     ✓ dplyr   1.0.7
    ## ✓ tidyr   1.1.3     ✓ stringr 1.4.0
    ## ✓ readr   2.0.1     ✓ forcats 0.5.1

    ## ── Conflicts ────────────────────────────────────────── tidyverse_conflicts() ──
    ## x dplyr::filter() masks stats::filter()
    ## x dplyr::lag()    masks stats::lag()

``` r
library(httr)
library(xml2)
library(stringr)
```

# Question 1: How many sars-cov-2 papers?

``` r
if (knitr::is_html_output(excludes = "gfm")){}
```

``` r
# Downloading the website
website <- xml2::read_html("https://pubmed.ncbi.nlm.nih.gov/?term=sars-cov-2")

# Finding the counts
counts <- xml2::xml_find_first(website, "/html/body/main/div[9]/div[2]/div[2]/div[1]/span")

# Turning it into text
counts <- as.character(counts)

# Extracting the data using regex
stringr::str_extract(counts, "[0-9,]+")
```

    ## [1] "114,592"

# Question 2: Academic publications on COVID19 and Hawaii

``` r
library(httr)
query_ids <- GET(
  url   = "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi",
  query = list(
    db="pubmed",
    term="covid19 hawaii",
    retmax=1000)
)

# Extracting the content of the response of GET
ids <- httr::content(query_ids)
ids
```

    ## {xml_document}
    ## <eSearchResult>
    ## [1] <Count>150</Count>
    ## [2] <RetMax>150</RetMax>
    ## [3] <RetStart>0</RetStart>
    ## [4] <IdList>\n  <Id>34562997</Id>\n  <Id>34559481</Id>\n  <Id>34545941</Id>\n ...
    ## [5] <TranslationSet>\n  <Translation>\n    <From>covid19</From>\n    <To>"cov ...
    ## [6] <TranslationStack>\n  <TermSet>\n    <Term>"covid-19"[MeSH Terms]</Term>\ ...
    ## [7] <QueryTranslation>("covid-19"[MeSH Terms] OR "covid-19"[All Fields] OR "c ...
