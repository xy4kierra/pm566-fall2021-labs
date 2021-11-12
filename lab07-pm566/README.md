Lab 07 - Web scraping and Regular Expressions-pm566
================
Xiaoyu Zhu
10/8/2021

# Lab description

In this lab, we will be working with the NCBI API to make queries and
extract information using XML and regular expressions. For this lab, we
will be using the httr, xml2, and stringr R packages.

This markdown document should be rendered using github\_document
document.

# Question 1: How many sars-cov-2 papers?

Build an automatic counter of sars-cov-2 papers using PubMed. You will
need to apply XPath as we did during the lecture to extract the number
of results returned by PubMed in the following web address:
<https://pubmed.ncbi.nlm.nih.gov/?term=sars-cov-2>

``` r
if (knitr::is_html_output(excludes = "gfm")){}
```

``` r
# Downloading the website
website <- xml2::read_html("https://pubmed.ncbi.nlm.nih.gov/?term=sars-cov-2")
# Finding the counts
counts <- xml2::xml_find_first(website, "/html/body/main/div[9]/div[2]/div[2]/div[1]/div[1]")
# Turning it into text
counts <- as.character(counts)
# Extracting the data using regex
stringr::str_extract(counts, "[0-9,]+")
```

    ## [1] "122,330"

``` r
stringr::str_extract(counts, "[[:digit:],]+")
```

    ## [1] "122,330"

``` r
stringr::str_replace(counts, "[^[:digit:]]+([[:digit:]]+),([[:digit:]]+)[^[:digit:]]+", "\\1\\2")
```

    ## [1] "122330"

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
    ## [1] <Count>182</Count>
    ## [2] <RetMax>182</RetMax>
    ## [3] <RetStart>0</RetStart>
    ## [4] <IdList>\n  <Id>34757265</Id>\n  <Id>34744296</Id>\n  <Id>34739906</Id>\n ...
    ## [5] <TranslationSet>\n  <Translation>\n    <From>covid19</From>\n    <To>"cov ...
    ## [6] <TranslationStack>\n  <TermSet>\n    <Term>"covid-19"[MeSH Terms]</Term>\ ...
    ## [7] <QueryTranslation>("covid-19"[MeSH Terms] OR "covid-19"[All Fields] OR "c ...

# Question 3: Get details about the articles

``` r
# Turn the result into a character vector
ids <- as.character(ids)
# Find all the ids 
ids <- stringr::str_extract_all(ids, "<Id>[[:digit:]]+</Id>")[[1]]
# Remove all the leading and trailing <Id> </Id>. Make use of "|"
# stringr::str_remove_all(ids, "</?Id>")
ids <- stringr::str_remove_all(ids, "<Id>|</Id>")
head(ids)
```

    ## [1] "34757265" "34744296" "34739906" "34736373" "34736125" "34734403"

``` r
publications <- GET(
  url   = "https://eutils.ncbi.nlm.nih.gov/",
  path  = "entrez/eutils/efetch.fcgi",
  query = list(
    db = "pubmed",
    id = I(paste(ids, collapse=",")),
    retmax = 1000,
    rettype = "abstract"
    )
)
# Turning the output into character vector
publications <- httr::content(publications)
publications_txt <- as.character(publications)
```

# Question 4: Distribution of universities, schools, and departments

Using the function stringr::str\_extract\_all() applied on
publications\_txt, capture all the terms of the form:

University of … … Institute of … Write a regular expression that
captures all such instances

``` r
library(stringr)
institution <- stringr::str_extract_all(
  str_to_lower(publications_txt),
  "[[:alpha:]-]+university|university\\s+of\\s+(southern|new|northern|the)?\\s*[[:alpha:]-]+|[[:alpha:]-]+\\s+institute\\s+of\\s+[[:alpha:]-]+"
  ) 

# institution <- unlist(institution)
table(institution)
```

    ## institution
    ##        australian institute of tropical       beijing institute of pharmacology 
    ##                                      15                                       2 
    ##              berlin institute of health              broad institute of harvard 
    ##                                       4                                       2 
    ##               cancer institute of emory                 cancer institute of new 
    ##                                       2                                       1 
    ##           genome institute of singapore    graduate institute of rehabilitation 
    ##                                       1                                       3 
    ##         health institute of montpellier          heidelberg institute of global 
    ##                                       1                                       1 
    ##                   i institute of marine            leeds institute of rheumatic 
    ##                                       1                                       2 
    ##   massachusetts institute of technology          medanta institute of education 
    ##                                       1                                       1 
    ## mediterranean institute of oceanography                 mgm institute of health 
    ##                                       1                                       1 
    ##       monterrey institute of technology           national institute of allergy 
    ##                                       1                                       1 
    ##     national institute of biostructures     national institute of environmental 
    ##                                       1                                       3 
    ##            national institute of public        national institute of technology 
    ##                                       1                                       1 
    ##        nordic institute of chiropractic               research institute of new 
    ##                                       1                                       4 
    ##      research institute of tuberculosis             the institute of biomedical 
    ##                                       2                                       1 
    ##               the institute of medicine                   university of alberta 
    ##                                       1                                       2 
    ##                   university of applied                   university of arizona 
    ##                                       1                                       5 
    ##                  university of arkansas                     university of basel 
    ##                                       7                                       8 
    ##                     university of benin                  university of botswana 
    ##                                       1                                       1 
    ##                  university of bradford                   university of bristol 
    ##                                       1                                       4 
    ##                   university of british                   university of calgary 
    ##                                       4                                       1 
    ##                university of california                   university of chicago 
    ##                                      66                                      11 
    ##                university of cincinnati                  university of colorado 
    ##                                       9                                       3 
    ##               university of connecticut                university of copenhagen 
    ##                                       1                                       1 
    ##                   university of córdoba                 university of education 
    ##                                       1                                       1 
    ##                    university of exeter                   university of florida 
    ##                                       1                                       5 
    ##                   university of granada                     university of haifa 
    ##                                       2                                       1 
    ##                     university of hawai                    university of hawaii 
    ##                                     169                                     189 
    ##              university of hawaii-manoa                    university of health 
    ##                                       2                                       8 
    ##                      university of hong                  university of honolulu 
    ##                                       1                                       3 
    ##                  university of illinois                      university of iowa 
    ##                                       1                                       4 
    ##                      university of juiz                    university of kansas 
    ##                                       4                                       2 
    ##                  university of kentucky                  university of lausanne 
    ##                                       1                                       1 
    ##                     university of leeds                university of louisville 
    ##                                       2                                       1 
    ##                    university of malaya                  university of maryland 
    ##                                       2                                       8 
    ##                   university of medical                  university of medicine 
    ##                                       2                                       3 
    ##                 university of melbourne                     university of miami 
    ##                                       1                                       2 
    ##                  university of michigan                 university of minnesota 
    ##                                       8                                       1 
    ##                    university of murcia                  university of nebraska 
    ##                                       1                                       5 
    ##                    university of nevada               university of new england 
    ##                                       1                                       1 
    ##                 university of new south                  university of new york 
    ##                                       3                                       3 
    ##       university of new york-university                     university of north 
    ##                                       1                                       2 
    ##                   university of ontario                      university of oslo 
    ##                                       1                                       6 
    ##                    university of oxford                   university of palermo 
    ##                                       9                                       1 
    ##                     university of paris              university of pennsylvania 
    ##                                       1                                      64 
    ##                university of pittsburgh                     university of porto 
    ##                                      13                                       2 
    ##                    university of puerto                       university of rio 
    ##                                       3                                       1 
    ##                 university of rochester                       university of sao 
    ##                                       4                                       2 
    ##                   university of science                 university of singapore 
    ##                                      13                                       1 
    ##                     university of south       university of southern california 
    ##                                       4                                      21 
    ##          university of southern denmark                    university of sydney 
    ##                                       1                                       1 
    ##                university of technology                     university of texas 
    ##                                       3                                       7 
    ##                university of the health           university of the philippines 
    ##                                      16                                       1 
    ##                   university of toronto                    university of toulon 
    ##                                       4                                       1 
    ##                  university of tübingen                      university of utah 
    ##                                       3                                       4 
    ##                university of washington                 university of wisconsin 
    ##                                       6                                       3 
    ##                      university of york  zoo-prophylactic institute of southern 
    ##                                       1                                       2

Repeat the exercise and this time focus on schools and departments in
the form of

School of … Department of … And tabulate the results

``` r
schools_and_deps <- str_extract_all(
  str_to_lower(publications_txt),
  "[[:alpha:]-]+\\s+school\\s+of\\s+(the)?\\s*[[:alpha:]-]+\\s|[[:alpha:]-]+\\s+department\\s+of\\s+[[:alpha:]-]+\\s"
  )
table(schools_and_deps)
```

    ## schools_and_deps
    ## abramson department of rehabilitation                     and school of life  
    ##                                      1                                      1 
    ##              burns school of medicine       carilion department of emergency  
    ##                                      7                                      1 
    ##                 chan school of public                cizik school of nursing  
    ##                                     19                                      2 
    ##             fielding school of public      florida department of agriculture  
    ##                                      8                                      1 
    ##             from department of health              geffen school of medicine  
    ##                                      1                                      6 
    ##             geisel school of medicine        government department of health  
    ##                                      2                                      2 
    ##           grossman school of medicine                harris school of public  
    ##                                      2                                      8 
    ##        hawaii department of education               hopkins school of public  
    ##                                      2                                      1 
    ##              icahn school of medicine                keck school of medicine  
    ##                                      9                                      8 
    ##               medical school of brown                 neill school of public  
    ##                                      2                                      2 
    ##     nuffield department of population                      s school of ocean  
    ##                                      3                                      1 
    ##            silberman school of social             state department of health  
    ##                                      2                                      2 
    ##         states department of veterans        the department of communication  
    ##                                      1                                      1 
    ##        the department of epidemiology           the department of preventive  
    ##                                      1                                      8 
    ##                  the school of public              thompson school of social  
    ##                                      1                                     29 
    ##         university school of medicine                us department of health  
    ##                                      2                                      1 
    ##             us department of veterans              uthealth school of public  
    ##                                      7                                      2 
    ##        zealand department of internal  
    ##                                     22

# Question 5: Form a database

We want to build a dataset which includes the title and the abstract of
the paper. The title of all records is enclosed by the HTML tag
ArticleTitle, and the abstract by Abstract.

Before applying the functions to extract text directly, it will help to
process the XML a bit. We will use the xml2::xml\_children() function to
keep one element per id. This way, if a paper is missing the abstract,
or something else, we will be able to properly match PUBMED IDS with
their corresponding records.

``` r
pub_char_list <- xml2::xml_children(publications)
pub_char_list <- sapply(pub_char_list, as.character)
```

``` r
abstracts <- stringr::str_extract(pub_char_list, "<Abstract>(\\n|.)+</Abstract>")
abstracts <- stringr::str_remove_all(abstracts, "</?[[:alnum:]]+>")
abstracts <- stringr::str_replace_all(abstracts, "\\s+", " ")
table(is.na(abstracts))
```

How many of these don’t have an abstract? Answer: There are 9 articles
without abstracts.

Now, the title

``` r
titles <- str_extract(pub_char_list, "<ArticleTitle>(\\n|.)+</ArticleTitle>")
titles <- str_remove_all(titles, "</?[[:alnum:]]+>")
titles <- str_replace_all(titles, "\\s+", " ")
table(is.na(titles))
```

There are no articles without titles.

Finally, put everything together into a single `data.frame` and use
`knitr::kable` to print the results

``` r
database <- data.frame(
  PubMedID  = ids,
  Title     = titles,
  Abstracts = abstracts
)
knitr::kable(database)
```

Done! Knit the document, commit, and push.

## Final Pro Tip (optional)

You can still share the HTML document on github. You can include a link
in your `README.md` file as the following:

``` md
View [here](https://ghcdn.rawgit.org/:user/:repo/:tag/:file)
```

For example, if we wanted to add a direct link the HTML page of lecture
7, we could do something like the following:

``` md
View week7-lab[here](https://ghcdn.rawgit.org/lysethan/PM566-labs/master/week7/week7-lab.html
```
