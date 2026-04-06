
<!-- README.md is generated from README.Rmd. Please edit that file -->

For the \#RStats using peasants

using {huggingfaceR}

``` r
huggingfaceR::hf_load_dataset(dataset = "OpenMed/pgc-schizophrenia", split = "train", limit = 1000)
#> # A tibble: 1,000 × 13
#>    snpid      hg18chr      bp a1    a2       or     se  pval  info   ngt
#>    <chr>        <int>   <int> <chr> <chr> <dbl>  <dbl> <dbl> <dbl> <int>
#>  1 rs10907175       1 1120590 A     C     1.01  0.0354 0.692 0.992    13
#>  2 rs2887286        1 1145994 T     C     0.986 0.0278 0.618 0.999    17
#>  3 rs11260562       1 1155173 A     G     1.000 0.0466 0.996 0.928    11
#>  4 rs7519837        1 1500664 T     C     1.01  0.0226 0.541 0.982    13
#>  5 rs12126768       1 1767950 T     G     1.03  0.027  0.349 0.993     5
#>  6 rs10907192       1 1782971 A     G     0.994 0.0474 0.903 0.962     3
#>  7 rs6603805        1 1811485 A     G     1.01  0.0233 0.795 0.984     0
#>  8 rs908742         1 2023116 A     G     0.983 0.0227 0.439 0.913    10
#>  9 rs3107147        1 2038589 A     G     1.03  0.0211 0.229 0.990     0
#> 10 rs3128309        1 2095738 A     G     1.03  0.0338 0.400 1.00     13
#> # ℹ 990 more rows
#> # ℹ 3 more variables: `_source_file` <chr>, .dataset <chr>, .split <chr>
```

Or {duckdb} + base R + {jsonlite}

``` r
DatasetParquetUrlList <- function(repo = "OpenMed/pgc-schizophrenia", dataset = "scz2011", split = "train") {
  # curl -X GET \
  #   "https://huggingface.co/api/datasets/OpenMed/pgc-schizophrenia/parquet/scz2011/train"
  urlList <- readLines(
    paste0(
      "https://huggingface.co/api/datasets/",
      repo,
      "/parquet/", dataset, "/", split
    ),
    warn = FALSE
  ) |>
    jsonlite::fromJSON(simplifyVector = FALSE) |>
    unlist()
  urlList
}

DatasetParquetUrlList() |> head()
#> [1] "https://huggingface.co/api/datasets/OpenMed/pgc-schizophrenia/parquet/scz2011/train/0.parquet"
#> [2] "https://huggingface.co/api/datasets/OpenMed/pgc-schizophrenia/parquet/scz2011/train/1.parquet"
#> [3] "https://huggingface.co/api/datasets/OpenMed/pgc-schizophrenia/parquet/scz2011/train/2.parquet"
#> [4] "https://huggingface.co/api/datasets/OpenMed/pgc-schizophrenia/parquet/scz2011/train/3.parquet"
#> [5] "https://huggingface.co/api/datasets/OpenMed/pgc-schizophrenia/parquet/scz2011/train/4.parquet"
#> [6] "https://huggingface.co/api/datasets/OpenMed/pgc-schizophrenia/parquet/scz2011/train/5.parquet"

OpenRemoteParquetView <- function(
  urlList = DatasetParquetUrlList(),
  view_name = "embeddings",
  db_path = tempfile(fileext = ".duckdb"),
  unify_schemas = FALSE
) {
  drv <- duckdb::duckdb()
  if (is.null(db_path)) {
    conn <- DBI::dbConnect(drv)
  } else {
    conn <- DBI::dbConnect(drv, dbdir = db_path)
  }

  DBI::dbExecute(conn, "INSTALL httpfs; LOAD httpfs;")

  sql <- sprintf(
    "CREATE OR REPLACE TEMPORARY VIEW %s AS SELECT * FROM parquet_scan(%s, UNION_BY_NAME=%s)",
    view_name,
    paste0("['", paste(urlList, collapse = "','"), "']"),
    tolower(unify_schemas)
  )
  DBI::dbExecute(conn, sql)

  list(
    conn = conn,
    tbl = dplyr::tbl(conn, view_name)
  )
}

remote <- OpenRemoteParquetView()
remote$tbl
#> # Source:   table<embeddings> [?? x 11]
#> # Database: DuckDB 1.5.1 [root@Linux 6.8.0-78-generic:R 4.5.2//tmp/Rtmpw4h27r/file136ae7cc41a4a.duckdb]
#>    snpid      hg18chr      bp a1    a2       or     se  pval  info   ngt
#>    <chr>        <dbl>   <dbl> <chr> <chr> <dbl>  <dbl> <dbl> <dbl> <dbl>
#>  1 rs10907175       1 1120590 A     C     1.01  0.0354 0.692 0.992    13
#>  2 rs2887286        1 1145994 T     C     0.986 0.0278 0.618 0.999    17
#>  3 rs11260562       1 1155173 A     G     1.000 0.0466 0.996 0.928    11
#>  4 rs7519837        1 1500664 T     C     1.01  0.0226 0.541 0.982    13
#>  5 rs12126768       1 1767950 T     G     1.03  0.027  0.349 0.993     5
#>  6 rs10907192       1 1782971 A     G     0.994 0.0474 0.903 0.962     3
#>  7 rs6603805        1 1811485 A     G     1.01  0.0233 0.795 0.984     0
#>  8 rs908742         1 2023116 A     G     0.983 0.0227 0.439 0.913    10
#>  9 rs3107147        1 2038589 A     G     1.03  0.0211 0.229 0.990     0
#> 10 rs3128309        1 2095738 A     G     1.03  0.0338 0.400 1.00     13
#> # ℹ more rows
#> # ℹ 1 more variable: `_source_file` <chr>
```

{duckdb} + duckhts for liftover to GRCh37

``` r
DBI::dbExecute(remote$conn, "INSTALL duckhts FROM community;")
#> [1] 0
DBI::dbExecute(remote$conn, "LOAD duckhts;")
#> [1] 0

lifted_scz <- DBI::dbGetQuery(
  remote$conn,
  paste(
    "SELECT src_chrom, src_pos, dest_chrom, dest_pos, mapped, reject_reason, note",
    "FROM duckdb_liftover(",
    "'(SELECT CAST(hg18chr AS VARCHAR) AS chrom, bp AS pos",
    "FROM embeddings",
    "WHERE hg18chr IS NOT NULL AND bp IS NOT NULL",
    "LIMIT 1000) AS scz',",
    "'chrom',",
    "'pos',",
    "chain_path := '/root/GRCh37/hg18ToHg19.over.chain.gz',",
    "dst_fasta_ref := '/root/GRCh37/human_g1k_v37.fasta'",
    ")"
  )
)

dplyr::as_tibble(lifted_scz) |>
  dplyr::slice_head(n = 10)
#> # A tibble: 10 × 7
#>    src_chrom src_pos dest_chrom dest_pos mapped reject_reason note 
#>    <chr>       <dbl> <chr>         <dbl> <lgl>  <chr>         <chr>
#>  1 1         3652705 chr1        3662845 TRUE   <NA>          <NA> 
#>  2 1         3729372 chr1        3739512 TRUE   <NA>          <NA> 
#>  3 1         4124418 chr1        4224558 TRUE   <NA>          <NA> 
#>  4 1         4304274 chr1        4404414 TRUE   <NA>          <NA> 
#>  5 1         4373862 chr1        4474002 TRUE   <NA>          <NA> 
#>  6 1         4534838 chr1        4634978 TRUE   <NA>          <NA> 
#>  7 1         4638538 chr1        4738678 TRUE   <NA>          <NA> 
#>  8 1         4731180 chr1        4831320 TRUE   <NA>          <NA> 
#>  9 1         4767710 chr1        4867850 TRUE   <NA>          <NA> 
#> 10 1         4931421 chr1        5031561 TRUE   <NA>          <NA>
```
