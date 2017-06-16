patentsview
================

> An R Client to the PatentsView API

[![Linux Build Status](https://travis-ci.org/crew102/patentsview.svg?branch=master)](https://travis-ci.org/crew102/patentsview) [![AppVeyor Build Status](https://ci.appveyor.com/api/projects/status/github/crew102/patentsview?branch=master&svg=true)](https://ci.appveyor.com/project/crew102/patentsview)

Installation
------------

``` r
devtools::install_github("crew102/patentsview")
```

Basic usage
-----------

The [PatentsView API](http://www.patentsview.org/api/doc.html) provides an interface to a disambiguated version of USPTO. The `patentsview` R package provides one main function, `search_pv()`, to make it easy to interact with that API. Let's take a look:

``` r
library(patentsview)

search_pv(query = '{"_gte":{"patent_date":"2007-01-01"}}',
          endpoint = "patents")
#> $data
#> #### A list with a single data frame on the patent data level:
#> 
#> List of 1
#>  $ patents:'data.frame': 25 obs. of  3 variables:
#>   ..$ patent_id    : chr [1:25] "7155746" ...
#>   ..$ patent_number: chr [1:25] "7155746" ...
#>   ..$ patent_title : chr [1:25] "Anti-wicking protective workwear and methods of making and using same" ...
#> 
#> $query_results
#> #### Distinct entity counts across all downloadable pages of output:
#> 
#> total_patent_count = 100,000
```

This call to `search_pv()` sends our query to the "patents" API endpoint. The API has 7 different endpoints, corresponding to 7 different entity types (assignees, CPC subsections, inventors, locations, NBER subcategories, patents, and USPC main classes).<sup><a href="#fn1" id="ref1">1</a></sup> We filtered our results using the API's [query language](http://www.patentsview.org/api/query-language.html) which uses JSON, though we could have used pure R functions (described in the [writing queries vignette](https://github.com/crew102/patentsview/blob/master/vignettes/writing-queries.Rmd)) instead.

Fields
------

Each endpoint has a different set of *queryable* and *retrievable* fields. Queryable fields are those that you can include in your query (e.g., `patent_date` shown above). Retrievable fields are those that you can get data on. In the above query we didn't specify which fields we wanted to retrieve, so we were given a default set of fields. You can specify which fields you want to retrieve using the `fields` argument:

``` r
search_pv(query = '{"_gte":{"patent_date":"2007-01-01"}}',
          endpoint = "patents", 
          fields = c("patent_number", "patent_title"))
#> $data
#> #### A list with a single data frame on the patent data level:
#> 
#> List of 1
#>  $ patents:'data.frame': 25 obs. of  2 variables:
#>   ..$ patent_number: chr [1:25] "7155746" ...
#>   ..$ patent_title : chr [1:25] "Anti-wicking protective workwear and methods of making and using same" ...
#> 
#> $query_results
#> #### Distinct entity counts across all downloadable pages of output:
#> 
#> total_patent_count = 100,000
```

To list all of the retrievable fields for a given endpoint, use `get_fields`:

``` r
retrvble_flds <- get_fields(endpoint = "patents")
head(retrvble_flds)
#> [1] "app_country"       "app_date"          "app_number"       
#> [4] "app_type"          "appcit_app_number" "appcit_category"
```

You can see an endpoint's list of queryable and retrievable fields by viewing the endpoint's online field list (e.g., the [inventor field list table](http://www.patentsview.org/api/inventor.html#field_list)). Note the "Query" column in this table, which lets you know whether the field is both queryable and retrievable (Query = Y), or just retrievable (Query = N). A list of all of the queryable and retrievable fields are also listed in the `fieldsdf` data frame, which you can load using `data("fieldsdf")` or `View(patentsview::fieldsdf)`.

Writing queries
---------------

The PatentsView query syntax is documented on their [query language page](http://www.patentsview.org/api/query-language.html#query_string_format).<sup><a href="#fn2" id="ref2">2</a></sup> It can be difficult to get your query right if you're writing it by hand (i.e., just writing the query in a string like `'{"_gte":{"patent_date":"2007-01-01"}}'`). The `patentsview` package comes with a simple domain specific language (DSL) to make writing queries a breeze. I recommend using this DSL for all but the most basic queries, especially if you're encountering errors and don't understand why. Check out the [writing queries vignette](https://github.com/crew102/patentsview/blob/master/vignettes/writing-queries.Rmd) for more details...We can rewrite our query using this DSL as:

``` r
qry_funs$gte(patent_date = "2007-01-01")
#> {"_gte":{"patent_date":"2007-01-01"}}
```

More complex queries are also possible:

``` r
with_qfuns(
  and(
    gte(patent_date = "2007-01-01"),
    text_phrase(patent_abstract = c("computer program", "dog leash"))
  )
)
#> {"_and":[{"_gte":{"patent_date":"2007-01-01"}},{"_or":[{"_text_phrase":{"patent_abstract":"computer program"}},{"_text_phrase":{"patent_abstract":"dog leash"}}]}]}
```

Paginated responses
-------------------

By default, `search_pv()` returns 25 records per page and only gives you the first page of results. I suggest using these defaults while you're figuring out the details of your request, such as the query syntax you want to use and the fields you want returned. Once you have those items finalized, you can use the `per_page` argument to download up to 10,000 records per page. You can also choose which page of results you want with the `page` argument:

``` r
search_pv(query = qry_funs$eq(inventor_last_name = "chambers"),
          page = 2, per_page = 150) # gets records 150 - 300
#> $data
#> #### A list with a single data frame on the patent data level:
#> 
#> List of 1
#>  $ patents:'data.frame': 150 obs. of  3 variables:
#>   ..$ patent_id    : chr [1:150] "4408568" ...
#>   ..$ patent_number: chr [1:150] "4408568" ...
#>   ..$ patent_title : chr [1:150] "Furnace wall ash monitoring system" ...
#> 
#> $query_results
#> #### Distinct entity counts across all downloadable pages of output:
#> 
#> total_patent_count = 1,812
```

You can download all pages of output in one call by setting `all_pages = TRUE`. This will set `per_page` equal to 10,000 and loop over all pages of output (downloading up to 10 pages, or 100,000 records total):

``` r
search_pv(query = qry_funs$eq(inventor_last_name = "chambers"),
          all_pages = TRUE)
#> $data
#> #### A list with a single data frame on the patent data level:
#> 
#> List of 1
#>  $ patents:'data.frame': 1812 obs. of  3 variables:
#>   ..$ patent_id    : chr [1:1812] "3931611" ...
#>   ..$ patent_number: chr [1:1812] "3931611" ...
#>   ..$ patent_title : chr [1:1812] "Program event recorder and data processing system" ...
#> 
#> $query_results
#> #### Distinct entity counts across all downloadable pages of output:
#> 
#> total_patent_count = 1,812
```

Entity counts
-------------

Our last two calls to `search_pv()` gave the same value for `total_patent_count`, even though we got a lot more data from the second call. This is because the entity counts returned by the API refer to the number of distinct entities across all *downloadable pages of output*, not just the page that was returned. Downloadable pages of output is an important phrase here, as the API limits us to 100,000 records per query. For example, we got `total_patent_count = 100,000` when we searched for patents published on or after 2007, even though there are way more than 100,000 patents that have been published since 2007. See the FAQs below for details on how to overcome the 100,000 record restriction.

By default, **PatentsView uses disambiguted versions of assignees, inventors, and locations, instead of raw data.** For example, let's say you search for all inventors whose first name is "john." The PatentsView API is going to return all of the inventors who have a preferred first name of john (as per the disambiguation results), which may not necessarily match their raw first name. You could be getting back inventors whose first name is, say, "jonathan," "johnn," or even "john jay." You can search on the raw inventor names instead of the preferred names by using the fields starting with "raw" in your query (e.g., `rawinventor_first_name`). Note that the assignee and location raw data fields are not currently being offered by the API. To see the methods behind the disambiguation process, see the [PatentsView Inventor Disambiguation Technical Workshop website](http://www.patentsview.org/workshop/)

7 endpoints for 7 entities
--------------------------

We can get similar data from the 7 endpoints. For example, the following two calls differ only in the endpoint that is chosen:

``` r
# Here we are using the patents endpoint
search_pv(query = qry_funs$eq(inventor_last_name = "chambers"), 
          endpoint = "patents", 
          fields = c("patent_number", "inventor_last_name", 
                     "assignee_organization"))
#> $data
#> #### A list with a single data frame (with list column(s) inside) on the patent data level:
#> 
#> List of 1
#>  $ patents:'data.frame': 25 obs. of  3 variables:
#>   ..$ patent_number: chr [1:25] "3931611" ...
#>   ..$ inventors    :List of 25
#>   ..$ assignees    :List of 25
#> 
#> $query_results
#> #### Distinct entity counts across all downloadable pages of output:
#> 
#> total_patent_count = 1,812
```

``` r
# While here we are using the assignees endpoint
search_pv(query = qry_funs$eq(inventor_last_name = "chambers"), 
          endpoint = "assignees", 
          fields = c("patent_number", "inventor_last_name", 
                     "assignee_organization"))
#> $data
#> #### A list with a single data frame (with list column(s) inside) on the assignee data level:
#> 
#> List of 1
#>  $ assignees:'data.frame':   25 obs. of  3 variables:
#>   ..$ assignee_organization: chr [1:25] "American Printing Components, LLC" ...
#>   ..$ patents              :List of 25
#>   ..$ inventors            :List of 25
#> 
#> $query_results
#> #### Distinct entity counts across all downloadable pages of output:
#> 
#> total_assignee_count = 467
```

Your choice of endpoint determines two things:

1.  **Which entity your query is applied to.** For the first call shown above, the API searched for *patents* that have at least one inventor on them with the last name of "chambers." For the second call, the API searched for *assignees* that were assigned a patent that has at least one inventor on it with the last name of "chambers."

2.  **The structure of the data frame that is returned.** The first call (which was to the patents endpoint) gave us a data frame on the *patent level*, meaning that each row corresponded to a different patent. Fields that were not on the patent level (e.g., `inventor_last_name`) were returned to us in list columns, named after the subentity that the field belongs to (e.g., the `inventors` subentity).<sup><a href="#fn3" id="ref3">3</a></sup> The second call gave us a data frame on the *assignee level*, meaning that each row corresponded to a different assignee.

Examples
--------

Which patents have been cited by more than 500 US patents?

``` r
search_pv(query = qry_funs$gt(patent_num_cited_by_us_patents = 500))
#> $data
#> #### A list with a single data frame on the patent data level:
#> 
#> List of 1
#>  $ patents:'data.frame': 25 obs. of  3 variables:
#>   ..$ patent_id    : chr [1:25] "3946398" ...
#>   ..$ patent_number: chr [1:25] "3946398" ...
#>   ..$ patent_title : chr [1:25] "Method and apparatus for recording with writing fluids and drop projection means therefor" ...
#> 
#> $query_results
#> #### Distinct entity counts across all downloadable pages of output:
#> 
#> total_patent_count = 2,436
```

How many distinct inventors (disambiguated) are represented by these highly-cited patents?

``` r
# Setting subent_cnts = TRUE will give us the subentity counts. Since inventors 
# are subentities for the patents endpoint, this means we will get their counts.
search_pv(query = qry_funs$gt(patent_num_cited_by_us_patents = 500),
          fields = c("patent_number", "inventor_id"), subent_cnts = TRUE)
#> $data
#> #### A list with a single data frame (with list column(s) inside) on the patent data level:
#> 
#> List of 1
#>  $ patents:'data.frame': 25 obs. of  2 variables:
#>   ..$ patent_number: chr [1:25] "3946398" ...
#>   ..$ inventors    :List of 25
#> 
#> $query_results
#> #### Distinct entity counts across all downloadable pages of output:
#> 
#> total_patent_count = 2,436, total_inventor_count = 4,223
```

What patents has Microsoft (disambiguated) published since 2010?

``` r
query <- with_qfuns(
  and(
    gte(patent_date = "2010-01-01"),
    contains(assignee_organization = "microsoft")
  )
)
search_pv(query = query)
#> $data
#> #### A list with a single data frame on the patent data level:
#> 
#> List of 1
#>  $ patents:'data.frame': 25 obs. of  3 variables:
#>   ..$ patent_id    : chr [1:25] "7643030" ...
#>   ..$ patent_number: chr [1:25] "7643030" ...
#>   ..$ patent_title : chr [1:25] "Method and system for efficiently evaluating and drawing NURBS surfaces for 3D graphics" ...
#> 
#> $query_results
#> #### Distinct entity counts across all downloadable pages of output:
#> 
#> total_patent_count = 20,629
```

Which assignees have an inventor whose last name contains "smith" (e.g., "smith", "johnson-smith")? Also, give me the patent data where those "smiths" occur.

``` r
# Get all possible retrievable assignee-level and patent-level fields for 
# the assignees endpoint:
asgn_pat_flds <- get_fields("assignees", c("assignees", "patents"))

# Ask the PatentsView API for these fields:
search_pv(query = qry_funs$contains(inventor_last_name = "smith"), 
          endpoint = "assignees", fields = asgn_pat_flds)
#> $data
#> #### A list with a single data frame (with list column(s) inside) on the assignee data level:
#> 
#> List of 1
#>  $ assignees:'data.frame':   25 obs. of  17 variables:
#>   ..$ assignee_first_name           : chr [1:25] NA ...
#>   ..$ assignee_first_seen_date      : chr [1:25] "2000-10-17" ...
#>   ..$ assignee_id                   : chr [1:25] "00043de7382082391622e725605f37b5" ...
#>   ..$ assignee_key_id               : chr [1:25] "20" ...
#>   ..$ assignee_last_name            : chr [1:25] NA ...
#>   ..$ assignee_last_seen_date       : chr [1:25] "2013-03-19" ...
#>   ..$ assignee_lastknown_city       : chr [1:25] "Bend" ...
#>   ..$ assignee_lastknown_country    : chr [1:25] "US" ...
#>   ..$ assignee_lastknown_latitude   : chr [1:25] "44.0582" ...
#>   ..$ assignee_lastknown_location_id: chr [1:25] "44.0581728|-121.3153096" ...
#>   ..$ assignee_lastknown_longitude  : chr [1:25] "-121.315" ...
#>   ..$ assignee_lastknown_state      : chr [1:25] "OR" ...
#>   ..$ assignee_organization         : chr [1:25] "UltraCard, Inc." ...
#>   ..$ assignee_total_num_inventors  : chr [1:25] "5" ...
#>   ..$ assignee_total_num_patents    : chr [1:25] "15" ...
#>   ..$ assignee_type                 : chr [1:25] "2" ...
#>   ..$ patents                       :List of 25
#> 
#> $query_results
#> #### Distinct entity counts across all downloadable pages of output:
#> 
#> total_assignee_count = 9,188
```

What are the top ten CPC subsections for patents funded by the DOE?

``` r
search_pv(query = qry_funs$contains(govint_org_name = "department of energy"), 
          endpoint = "cpc_subsections", 
          fields =  "cpc_total_num_patents",
          sort = c("cpc_total_num_patents" = "desc"), 
          per_page = 10)
#> $data
#> #### A list with a single data frame on the CPC subsection data level:
#> 
#> List of 1
#>  $ cpc_subsections:'data.frame': 10 obs. of  1 variable:
#>   ..$ cpc_total_num_patents: chr [1:10] "840412" ...
#> 
#> $query_results
#> #### Distinct entity counts across all downloadable pages of output:
#> 
#> total_cpc_subsection_count = 121
```

FAQ
---

#### I'm sure my query is well formatted and correct but I keep getting an error. What's the deal?

The API query syntax guidelines do not cover all of the API's behavior. Specifically, there are several things that you cannot do which are not documented on the API's webpage. The [writing queries vignette](https://github.com/crew102/patentsview/blob/master/vignettes/writing-queries.Rmd) has more details on this.

#### Does the API have any rate limiting/throttling controls?

Not at the moment.

#### How do I download more than 100,000 records?

Your best bet is to split your query into pieces based on dates, then concatenate the results together. For example, the below query would return more than 100,000 records for the patents endpoint:

``` r
query <- with_qfuns(
  text_any(patent_abstract = 'tool animal')
)
```

To download all of the records associated with this query, we could split it into two pieces and make two calls to `search_pv()`:

``` r
query_1a <- with_qfuns(
  and(
    text_any(patent_abstract = 'tool animal'),
    lte(patent_date = "2010-01-01")
  )
)

query_1b <- with_qfuns(
  and(
    text_any(patent_abstract = 'tool animal'),
    gt(patent_date = "2010-01-01")
  )
)
```

#### How do I access the data frames inside of the subentity list columns?

You can unnest the data frames from the list columns using `unnest_pv_data()`. This function creates a series of data frames that are like tables in a relational database. The data frames can be linked together using the primary key that you specify. For example, in this call our primary entity is the assignee, and the subentities include applications and "government interest statements":

``` r
res <- search_pv(query = qry_funs$contains(inventor_last_name = "smith"), 
                 endpoint = "assignees", 
                 fields = get_fields("assignees", c("assignees","applications", 
                                                    "gov_interests")))
res$data
#> #### A list with a single data frame (with list column(s) inside) on the assignee data level:
#> 
#> List of 1
#>  $ assignees:'data.frame':   25 obs. of  18 variables:
#>   ..$ assignee_first_name           : chr [1:25] NA ...
#>   ..$ assignee_first_seen_date      : chr [1:25] "2000-10-17" ...
#>   ..$ assignee_id                   : chr [1:25] "00043de7382082391622e725605f37b5" ...
#>   ..$ assignee_key_id               : chr [1:25] "20" ...
#>   ..$ assignee_last_name            : chr [1:25] NA ...
#>   ..$ assignee_last_seen_date       : chr [1:25] "2013-03-19" ...
#>   ..$ assignee_lastknown_city       : chr [1:25] "Bend" ...
#>   ..$ assignee_lastknown_country    : chr [1:25] "US" ...
#>   ..$ assignee_lastknown_latitude   : chr [1:25] "44.0582" ...
#>   ..$ assignee_lastknown_location_id: chr [1:25] "44.0581728|-121.3153096" ...
#>   ..$ assignee_lastknown_longitude  : chr [1:25] "-121.315" ...
#>   ..$ assignee_lastknown_state      : chr [1:25] "OR" ...
#>   ..$ assignee_organization         : chr [1:25] "UltraCard, Inc." ...
#>   ..$ assignee_total_num_inventors  : chr [1:25] "5" ...
#>   ..$ assignee_total_num_patents    : chr [1:25] "15" ...
#>   ..$ assignee_type                 : chr [1:25] "2" ...
#>   ..$ applications                  :List of 25
#>   ..$ gov_interests                 :List of 25
```

The `res$data` data frame has assignee-level vector columns (e.g., the vector `res$data$assignees$assignee_first_seen_date`) and subentity-level list columns (e.g., the list `res$data$assignees$applications`). The list columns have data frames nested inside them, which we can extract using `unnest_pv_data()`:

``` r
unnest_pv_data(data = res$data, pk = "assignee_id")
#> List of 3
#>  $ applications :'data.frame':   124 obs. of  5 variables:
#>   ..$ assignee_id: chr [1:124] "00043de7382082391622e725605f37b5" ...
#>   ..$ app_country: chr [1:124] "US" ...
#>   ..$ app_date   : chr [1:124] "1998-07-10" ...
#>   ..$ app_number : chr [1:124] "09113783" ...
#>   ..$ app_type   : chr [1:124] "09" ...
#>  $ gov_interests:'data.frame':   25 obs. of  8 variables:
#>   ..$ assignee_id                 : chr [1:25] "00043de7382082391622e725605f37b5" ...
#>   ..$ govint_contract_award_number: logi [1:25] NA ...
#>   ..$ govint_org_id               : logi [1:25] NA ...
#>   ..$ govint_org_level_one        : logi [1:25] NA ...
#>   ..$ govint_org_level_three      : logi [1:25] NA ...
#>   ..$ govint_org_level_two        : logi [1:25] NA ...
#>   ..$ govint_org_name             : logi [1:25] NA ...
#>   ..$ govint_raw_statement        : logi [1:25] NA ...
#>  $ assignees    :'data.frame':   25 obs. of  16 variables:
#>   ..$ assignee_id                   : chr [1:25] "00043de7382082391622e725605f37b5" ...
#>   ..$ assignee_first_name           : chr [1:25] NA ...
#>   ..$ assignee_first_seen_date      : chr [1:25] "2000-10-17" ...
#>   ..$ assignee_key_id               : chr [1:25] "20" ...
#>   ..$ assignee_last_name            : chr [1:25] NA ...
#>   ..$ assignee_last_seen_date       : chr [1:25] "2013-03-19" ...
#>   ..$ assignee_lastknown_city       : chr [1:25] "Bend" ...
#>   ..$ assignee_lastknown_country    : chr [1:25] "US" ...
#>   ..$ assignee_lastknown_latitude   : chr [1:25] "44.0582" ...
#>   ..$ assignee_lastknown_location_id: chr [1:25] "44.0581728|-121.3153096" ...
#>   ..$ assignee_lastknown_longitude  : chr [1:25] "-121.315" ...
#>   ..$ assignee_lastknown_state      : chr [1:25] "OR" ...
#>   ..$ assignee_organization         : chr [1:25] "UltraCard, Inc." ...
#>   ..$ assignee_total_num_inventors  : chr [1:25] "5" ...
#>   ..$ assignee_total_num_patents    : chr [1:25] "15" ...
#>   ..$ assignee_type                 : chr [1:25] "2" ...
```

Now we are left with a series of flat data frames, instead of a single data frame with data frames nested inside. These flat data frames can be joined together as needed via the primary key (`assignee_id`).

------------------------------------------------------------------------

<sup id="fn1">1</sup> You can use `get_endpoints()` to list the endpoint names as the API expects them to appear (e.g., `assignees`, `cpc_subsections`, `inventors`, `locations`, `nber_subcategories`, `patents`, and `uspc_mainclasses`)<sup><a href="#ref1">back</a></sup>

<sup id="fn2">2</sup> This webpage includes some details that are not relevant to the `query` argument in `search_pv`, such as the field list and sort parameter.<sup><a href="#ref2">back</a></sup>

<sup id="fn3">3</sup> You can unnest the data frames that are stored in the list columns using `unnest_pv_data()`. See the FAQs for details.<sup><a href="#ref3">back</a></sup>
[![ropensci\_footer](http://ropensci.org/public_images/github_footer.png)](http://ropensci.org)
