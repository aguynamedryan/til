## 2025-01-16

### Ruby\#then

- Ruby has a `then` method that allows me to do stuff pipeline style
- Coming back from a long time in R, this is a nice way to avoid nesting tons of calls
- There are apparently some gems that add syntax sugar and other means of doing this kind of thing, but this approach feels like a nice balance between idiomatic Ruby and R

```ruby
Sequel.function(:length, col)
  .then { |c| Sequel.function(:max, c)} # => Sequel.function(:max, Sequel.function(:length, col))
```

### Copilot is Awesome

- Admittedly, I don't do a lot of work in traditional IDEs or languages that support good autocomplete (*cough* Ruby)
- Recently switched back from LazyVim via terminals to VSCode to try out GitHub's free copilot offering
- Holy moley it is awesome
- Not just autocomplete...it feels like it reads my mind some times
- About 10% of the time it autocompletes something that is wrong, like making a variable name instead of a symbol
- But it saves me a lot of context switching by filling out what I'm planning to write so I can move on to the next thing I want to write

## 2024-12-20

- Ruby Sequel can put [order by in a function](https://sequel.jeremyevans.net/rdoc/classes/Sequel/SQL/Function.html) among other things
  - `Sequel.function(:foo, :a).order(:a, Sequel.desc(:b)) # foo(a ORDER BY a, b DESC)`

## 2024-12-19

- R can do "raw strings": `string <- r"(Hello, "quoted" world!)"`

## 2024-12-16

- MotherDuck can run [LLMs as a query](https://motherduck.com/blog/llm-data-pipelines-prompt-motherduck-dbt/?utm_campaign=MotherDuck%20News&utm_medium=email&_hsenc=p2ANqtz-8DpIu13Xos94QivieA_DNUJ16jnhqWlcBVtxCySaCZOiskF_ZXsCx4ihl178AbjEMREeby4qcwB1gMDVSygYECflP9Vg&_hsmi=338558250&utm_content=338505640&utm_source=hs_email)
- There's a site that [compares pricing and high-level features of AI models](https://countless.dev/)

## 2024-12-10

### AWS Athena Query Plans

- Trino can output a plan as Graphviz: `https://trino.io/docs/current/sql/explain.html`
  - `dbGetQuery(conn, paste0("explain ", read_file("tc_full_cte/tc_med_og_select_only.sql"))) |> dplyr::pull(\`Query Plan\`) |> paste0(collapse = "\n") |> message()`
  - The table names it puts out in AWS Athena are crazy-long, so we'll need to chop those down in order to render anything remotely useful

## 2024-11-20

### S7?

- Tidyverse team is [introducing YAF OOP system](https://www.tidyverse.org/blog/2024/11/s7-0-2-0/) to R using a combo of S3 and S4 called [S7](https://rconsortium.github.io/S7/index.html)

### nanoparquet

- Tidyverse team is [developing nanoparquet](https://www.tidyverse.org/blog/2024/06/nanoparquet-0-3-0/), but Ima let that bake for a while longer

### Presto Doesn't Have TEMPORARY TABLE/VIEW

- No present in the documentation
- [This SO post](https://stackoverflow.com/questions/73242331/how-to-create-a-temporary-table-in-presto-sql) says it's missing
- [This SO post](https://stackoverflow.com/questions/42563301/presto-create-table-with-with-queries) says to use CTEs instead

## 2024-11-19

### Output {} Using R's jsonlite

- [Found this](https://stackoverflow.com/a/20109959/14450422)
- TLDR: `setNames(list(), character(0)) |> toJSON()`

## 2024-11-18

### Overriding S3 Methods In R

- I needed to override some methods in the [`RPresto`](https://github.com/prestodb/RPresto) package
- I found [this SO suggestion](https://stackoverflow.com/a/72772582/14450422)
- I wrote this and saved it to zzz.R:

```r
# Presto does not support MEDIAN and QUANTILE directly
#
# RPresto issues an error if we try to call those functions in a mutate
# statement
#
# I have an acceptable SQL workaround for Presto
#
# I would like to avoid rewriting the dbplyr-based code I've written that uses
# these functions and instead use the workarounds
#
# I found two functions (sd, var) that were not exactly correctly handled by
# RPresto and I get warning messages about rm.na = TRUE needing to be set
#
# I found [this post](https://stackoverflow.com/a/72772582/14450422) on how to
# override S3 methods
#
# So this code takes the original RPresto translations and overwrites the four I
# needed changed, then creates a function that returns the updated translations
# and I override the original S3 method with the new function
RPresto_overrides <- function() {
  new_median <- purrr::partial(dbplyr:::sql_quantile("APPROX_PERCENTILE"), probs = 0.5)
  new_quantile <- dbplyr:::sql_quantile("APPROX_PERCENTILE")
  new_sd <- dbplyr:::sql_aggregate("STDDEV_SAMP", "sd")
  new_var <- dbplyr:::sql_aggregate("VAR_SAMP", "var")

  og_translator <- RPresto:::sql_translation.PrestoConnection
  allows_median_and_others <- function(con) {
    t <- og_translator(con)
    dbplyr::sql_variant(
      dbplyr::sql_translator(
        .parent = t[["scalar"]],
        median = new_median,
        quantile = new_quantile
       ),
      dbplyr:::sql_translator(
        .parent = t[["aggregate"]],
        median = new_median,
        quantile = new_quantile,
        sd = new_sd,
        var = new_var
      ),
      dbplyr:::sql_translator(
        .parent = t[["window"]],
        median = new_median,
        quantile = new_quantile,
        sd = new_sd,
        var = new_var
      )
    )
  }
  return(allows_median_and_others)
}

register_all_s3_methods <- function() {
  vctrs::s3_register("RPresto::sql_translation", "PrestoConnection", RPresto_overrides())
}

.onLoad = function(libname, pkgname) {
  register_all_s3_methods()
}
```

### ipynb Files are JSON

- And [here's the spec](https://ipython.org/ipython-doc/3/notebook/nbformat.html)
- And a [more uptodate version of the spec](https://nbformat.readthedocs.io/en/latest/format_description.html)

### Jupyter Notebooks Can Strip Output

- [Here's how](https://gist.github.com/parente/bfd7548ee08b6e377da85f8e4f88d6b8)
- TLDR: `jupyter nbconvert my_input_notebook.ipynb --to notebook --ClearOutputPreprocessor.enabled=True --stdout > my_output_notebook.ipynb`
