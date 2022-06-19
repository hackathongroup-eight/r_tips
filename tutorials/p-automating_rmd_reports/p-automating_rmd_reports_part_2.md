Automate R Markdown report generation - Part 2
================
Erika Duan
2022-06-19

-   [Introduction](#introduction)
-   [Step 1: Establish a project
    workflow](#step-1-establish-a-project-workflow)
    -   [Use consistent names](#use-consistent-names)
    -   [Environment reproducibility](#environment-reproducibility)
-   [Step 2: Create data ingestion R
    script](#step-2-create-data-ingestion-r-script)
-   [Step 3: Create data cleaning R
    script](#step-3-create-data-cleaning-r-script)
    -   [(Optional) Data version
        control](#optional-data-version-control)
-   [Step 3: Create an R Markdown report
    template](#step-3-create-an-r-markdown-report-template)
-   [Step 4: Create an R script for report
    automation](#step-4-create-an-r-script-for-report-automation)
-   [Step 5: Create a CI/CD pipeline using GitHub
    Actions](#step-5-create-a-cicd-pipeline-using-github-actions)
-   [Resources](#resources)

``` r
# Load required packages -------------------------------------------------------  
if (!require("pacman")) install.packages("pacman")
pacman::p_load(here,
               janitor,
               rsdmx,
               clock,
               tidyverse,
               renv)  
```

# Introduction

This tutorial follows from [part
1](https://github.com/erikaduan/r_tips/blob/master/tutorials/p-automating_rmd_reports/p-automating_rmd_reports_part_1.md),
which shows how to use parameter key-value pairs for automated reporting
in R.

Creating an automated reporting workflow requires the following setup:

1.  A place to store the code, data inputs and analytical outputs, as
    well as information about the virtual environment and R packages
    used.  
2.  A data ingestion script. Data version control can also be integrated
    immediately following this step.  
3.  A data cleaning script.  
4.  An R Markdown report template containing YAML parameter key-value
    pairs.  
5.  A report automation script which iterates across all analytical
    parameters of interest.  
6.  A YAML pipeline using GitHub Actions.

For this tutorial, I have created a separate GitHub repository named
[`abs_labour_force_report`](https://github.com/erikaduan/abs_labour_force_report)
to host an automated R reporting workflow using open ABS labour force
data. The rest of this tutorial will reference this GitHub repository.

# Step 1: Establish a project workflow

## Use consistent names

There is no universally best way to organise your project structure. I
recommend starting with a simple naming structure that everyone easily
understands. My
[`abs_labour_force_report`](https://github.com/erikaduan/abs_labour_force_report)
repository contains the folders `./code` to store my R scripts and Rmd
documents, `./data` to store my data, and `./output` to store my
analytical outputs.

<img src="../../figures/p-automating_rmd_reports-project_structure_basic.svg" width="60%" style="display: block; margin: auto;" />

**Note:** The `./data` folder contains subfolders `./data/raw_data` and
`./data/clean_data` to maintain separation between the raw (read only)
and clean dataset used for further analysis.

## Environment reproducibility

Besides the `code`, `data` and `output` directories, a reproducible
virtual environment also needs to be created to host the entire
workflow. In this tutorial, I have chosen to host my
[`abs_labour_force_report`](https://github.com/erikaduan/abs_labour_force_report)
workflow using `GitHub Actions`, where code is triggered by a YAML
pipeline stored in `./.github/workflows/main.yml`.

To execute my R code, I need to create a virtual environment which loads
my project R package dependencies and R scripts in order. R package
dependencies can be managed by the
[`renv`](https://rstudio.github.io/renv/articles/renv.html) package.

To do this, the following steps need to be implemented **after** you
have written all your R scripts:

1.  Install `renv` with `install.packages('renv')`.  
2.  Run `renv::init()` directly in the R console to create a new
    project-local environment with a private R library. This only needs
    to be done once per project as `renv::init()` amends the
    project-local `.Rprofile` with new code to load the private R
    library when an R session is started.  
3.  Use `install.packages("package")` and `remove.packages("package")`
    to keep installing or removing packages as required.  
4.  Run `renv::snapshot()` directly in the R console. This saves the
    state of your project R package dependencies to a `renv.lock`
    lockfile.  
5.  Each time your workflow is rerun inside a new virtual environment,
    the shell command `R -e 'renv::restore()'` needs to run in your YAML
    pipeline to restore your project R package dependencies from your
    `renv.lock` file.

<img src="../../figures/p-automating_rmd_reports-project_structure_advanced.svg" width="90%" style="display: block; margin: auto;" />

**Note:** An additional `.gitignore` file is automatically generated
inside `~/renv` to prevent project R packages and related `renv` files
from being committed into your remote project repository.

\#TODO

# Step 2: Create data ingestion R script

# Step 3: Create data cleaning R script

## (Optional) Data version control

Data version control can also be implemented as a step in the project
workflow, to track the version of data files being used. A popular way
to do this is by creating and committing data version pointers using the
open-source software
[`dvc`](https://dvc.org/blog/r-code-and-reproducible-model-development-with-dvc).

This step is currently not covered by this tutorial.

We need to create a single R script that:

1.  Downloads the raw dataset from its source location (i.e. from an URL
    or data API).  
2.  Saves the raw dataset.  
3.  Cleans the raw dataset and saves the clean dataset.

This setup allows us to automate future data extractions, assuming that
there are no changes to the data source (i.e. its URL or schema). We
would use the code below and save it as
[`abs_labour_force_report/code/01_extract_and_clean_data.R`](https://github.com/erikaduan/abs_labour_force_report/blob/main/code/01_extract_and_clean_data.R).

``` r
# Load required packages -------------------------------------------------------  
# Input package names
packages <- c("here",
              "readr",
              "stringr",
              "janitor",
              "rsdmx",
              "clock",
              "dplyr",
              "magrittr") 

installed_packages <- packages %in% rownames(installed.packages())

# Install new packages
if (any(installed_packages == FALSE)) {
  install.packages(packages[!installed_packages])
}

# Load new packages silently  
invisible(lapply(packages, library, character.only = TRUE))

# Connect to Labour Force API --------------------------------------------------
data_url <- "https://api.data.abs.gov.au/data/ABS,LF,1.0.0/M2+M1.2+1+3.1599.20+30.AUS.M?startPeriod=2019-01&dimensionAtObservation=AllDimensions"  

# Obtain data as tibble data frame ---------------------------------------------
labour_force <- readSDMX(data_url) %>%
  as_tibble() 

# Save raw data ----------------------------------------------------------------
write_csv(labour_force, here("data",
                             "raw_data",
                             "labour_force_raw.csv"))

# Clean data to produce YAML parameter friendly dataset ------------------------  
labour_force <- labour_force %>%
  clean_names() %>%
  filter (tsest == 20) %>% # Extract seasonally adjusted values  
  select(time_period,
         measure,
         sex,
         obs_value) 

# Rename measure and sex as strings   
labour_force <- labour_force %>% 
  mutate(measure = case_when(measure == "M1" ~ "full-time",
                             measure == "M2" ~ "part-time"),
         sex = case_when(sex == "1" ~ "male",
                         sex == "2" ~ "female",
                         sex == "3" ~ "all"))

# Convert time_period into a Date format and create a change variable 
labour_force <- labour_force %>% 
  group_by(measure, sex) %>% # Analyse for all subcategories of measure and sex  
  mutate(time_period = as.Date(paste0(time_period, "-01"), format = "%Y-%m-%d"),
         last_obs_value = lag(obs_value),
         change_obs_value = case_when(
           is.na(last_obs_value) ~ 0,
           TRUE ~ obs_value - last_obs_value)) %>%
  ungroup()

# Save clean data --------------------------------------------------------------
write_csv(labour_force, here("data",
                             "clean_data",
                             "labour_force_clean.csv"))
```

# Step 3: Create an R Markdown report template

The R Markdown template contains the R code and any additional markdown
or html code required for building the final report.

The only difference between a standard R Markdown report versus report
template is the absence of hard coded variables or visible code chunks
in the template. The report template should contain the minimal code
required to generate your report outputs (i.e. figures, tables and
summary text).

Example code from the R Markdown template located in
[`abs_labour_force_report/code/02_create_report_template.Rmd`](https://github.com/erikaduan/abs_labour_force_report/blob/main/code/02_create_report_template.Rmd)
is shown below.

``` r
# Plot data --------------------------------------------------------------------
# Fix y-axis between different reports 
y_max <- max(labour_force$change_obs_value) 
y_min <- min(labour_force$change_obs_value)   

labour_force %>% 
  filter(sex == params$sex, 
         measure == params$measure) %>%
  ggplot(aes(x = time_period, 
             y = change_obs_value)) +
  geom_line() + 
  scale_y_continuous(limits = c(y_min, y_max)) +
  geom_vline(xintercept = as.Date("2020-02-01"),
             colour = "firebrick",
             linetype = "dashed") +
  annotate("label",
           x = as.Date("2020-02-01"),
           y = y_max - 10,
           label = "COVID-19", color = "firebrick") +
  labs(title = paste("Labour force change for", params$sex, params$measure, "individuals"), 
       x = NULL,
       y = "Individuals (1000s)") +
  theme_bw() +
  theme(panel.grid.major.x = element_blank(),
        panel.grid.minor.x = element_blank(),
        panel.grid.major.y = element_line(linetype = "dotted"),
        panel.grid.minor.y = element_line(linetype = "dotted"),
        plot.title = element_text(hjust = 0.5))
```

**Note:** Always use default reporting values inside your R Markdown
YAML parameters i.e. `category: "all"`. This allows you to preview your
default report when testing your template code using `knit`.

# Step 4: Create an R script for report automation

To generate reports for all parameters of interest, we need to create an
R script with a for loop that performs the following actions:

1.  Creates a data frame containing all parameters of interest,
    extracted from your clean dataset.  
2.  Runs `render`, which inputs the report template and list of
    parameters to generate the final set of output files.

This step is shown in the code below, which is located in
[`abs_labour_force_report/code/03_automate_reports.R`](https://github.com/erikaduan/abs_labour_force_report/blob/main/code/03_automate_reports.R).

``` r
# Load required packages -------------------------------------------------------  
# Input package names
packages <- c("here",
              "readr",
              "stringr", 
              "clock",
              "dplyr",
              "ggplot2",
              "rmarkdown",
              "magrittr") 

installed_packages <- packages %in% rownames(installed.packages())

# Install new packages
if (any(installed_packages == FALSE)) {
  install.packages(packages[!installed_packages])
}

# Load new packages silently  
invisible(lapply(packages, library, character.only = TRUE))

# Load clean data --------------------------------------------------------------  
labour_force <- read_csv(here("data",
                              "clean_data",
                              "labour_force_clean.csv"))    

# Create for loop to automate report generation --------------------------------
# Create data frame of the dot product of all parameter values 
params_df <- expand.grid(unique(labour_force$sex), unique(labour_force$measure),
                         stringsAsFactors = FALSE)  

# Input template report and parameters to output all html reports
for (i in 1:nrow(params_df)) {
  rmarkdown::render(
    input = here("code",
                 "02_create_report_template.Rmd"),
    params = list(sex = params_df[i, 1],
                  measure = params_df[i, 2]),
    output_file = here("output",
                       glue::glue("{params_df[i, 1]}_{params_df[i, 2]}_report.md"))
  )
}
```

**Note:** Remember to save your output files as `.html` files if you
want to render html reports, and as `.Rmd` files if you want to render
GitHub documents with an accompanying `.Rmd` output.

# Step 5: Create a CI/CD pipeline using GitHub Actions

The ability to perform continuous integration and continuous delivery
(CI/CD) is a feature of all modern data platforms. CI/CD processes allow
simple to complex data jobs to be processed in response to automatic or
manual triggers.

The important components inside a CI/CD pipeline are:

-   Events: The condition which triggers a CI/CD pipeline to run
    i.e. when a data source is updated or when new code is committed.  
-   Jobs: The collection of tasks which are executed when an event is
    triggered. Multiple jobs can run sequentially or in parallel with
    each other.  
-   Workflows: A workflow is the entire automation process, comprised of
    individual or multiple jobs that are triggered by a specific event.
    In GitHub Actions, workflows are defined using a YAML file inside
    the `.github/workflows` directory.

Since I know that the ABS updates its labour force data once a month, I
am interested in creating a GitHub Actions workflow which reruns my code
at the end of every month. Creating this workflow requires the following
steps:

1.  **Recommended:** Make sure your workflow is contained in a separate
    GitHub repository. This allows you to create a separate `.Rproj`
    file, `.github/workflows` directory and project local `renv` setup
    specific to your workflow. My `abs_labour_force_report` GitHub
    repository is found
    [here](https://github.com/erikaduan/abs_labour_force_report).  
2.  Once the code inside your workflow is stable, open your R project
    file and use `renv::init()` to initialize a new project-local
    environment with a private R library. Run your code and then use
    `renv::snapshot()` to fix the versions of your required R packages.
    This creates an `renv.lock` file listing the different package
    versions used for your workflow.  
3.  Create a YAML template named
    `repository_name/.github/workflows/main.yml`. A quick way of doing
    this is to navigate to the Actions tab in your GitHub repository and
    to set up a basic workflow template yourself.

<img src="../../figures/p-automating_rmd_reports-github_actions_tab.png" width="90%" style="display: block; margin: auto;" />

4.  Create your GitHub Actions workflow by adapting your YAML file using
    templates provided by
    [`r-lib`](https://github.com/r-lib/actions/tree/master/setup-r). My
    own YAML template is derived from this [tutorial
    example](https://github.com/RMHogervorst/invertedushape/blob/main/.github/workflows/main.yml).

    ``` r
    # YAML workflow for abs_labour_force_report ------------------------------------
    name: ABS_labour_force_report
    on:
      # Workflow scheduled to run at 00:00 UTC on the 1st of every month.
      schedule:
    - cron:  "0 0 1 * *"

      # Workflow can also be run manually from the Actions tab
      workflow_dispatch:

    # Construct a job to load the runner, set up the R environment, run scripts and git commit
    jobs:
      run_report:
    runs-on: ubuntu-latest

    # Retrieve secrets from GitHub
    env:
        apikey: ${{ secrets.APIKEY}}
        apisecretkey: ${{ secrets.APISECRETKEY}}
        access_token: ${{ secrets.ACCESS_TOKEN}}
        access_token_secret: ${{ secrets.ACCESS_TOKEN_SECRET}}
        RENV_PATHS_ROOT: ~/.local/share/renv

    steps:
      # Checks out your repository under $GITHUB_WORKSPACE, so your job can access it
      - uses: actions/checkout@v2
      # Sets up pandoc which is required for knitting HTML reports  
      - uses: r-lib/actions/setup-pandoc@v2
        with:
          pandoc-version: '2.17.1' 

      # Set up R environment and install renv packages  
      - name: setup-r
        uses: r-lib/actions/setup-r@v1
        with:
          r-version: '4.1.2'
      - run: R -e 'install.packages("renv")'

      # Set up packages cache for future report reloads 
      - name: cache-packages
        uses: actions/cache@v1
        with:
           path: ${{ env.RENV_PATHS_ROOT }}
           key: ${{ runner.os }}-renv-${{ hashFiles('**/renv.lock') }}
           restore-keys: |-
              ${{ runner.os }}-renv-
      - run: sudo apt-get install -y --no-install-recommends libcurl4-openssl-dev
      # Use renv::restore() to install C++ dependencies and packages
      - run: R -e 'renv::restore()'

      # Execute R scripts
      - run: Rscript code/01_extract_and_clean_data.R
      - run: Rscript code/03_automate_reports.R  

      # Commit newly rendered reports into the repository 
      - name: commit-new-files
        run: |
          git config --local user.email "actions@github.com"
          git config --local user.name "GitHub Actions"
          git add --all
          git commit -am ":package: refresh and produce new report version"
          git push 
    ```

5.  Commit your GitHub Actions YAML workflow into your repository.
    Congratulations! You have now set up a simple CI/CD workflow using
    GitHub actions.

**Note:** Creating a cache for R packages is recommended as downloading
packages takes a long time to execute in GitHub Actions.

# Resources

-   A great [blog post](https://ptds.samorso.ch/tutorials/workflow/)
    containing useful advice for setting up a reproducible project
    workflow.  

-   A great [presentation](bit.ly/marvelRMD) and companion [blog
    post](https://themockup.blog/posts/2020-07-25-meta-rmarkdown/) by
    Thomas Mock on advanced R Markdown features.  

-   A useful [blog
    post](https://gabrieltanner.org/blog/an-introduction-to-github-actions)
    that summarises the features of GitHub Actions for CI/CD.  

-   Useful blog posts
    [here](https://blog.rmhogervorst.nl/blog/2020/09/24/running-an-r-script-on-a-schedule-gh-actions/)
    and
    [here](https://amitlevinson.com/blog/automated-plot-with-github-actions/),
    which describe how to use GitHub Actions to schedule R scripts.  

-   A [YouTube tutorial](https://www.youtube.com/watch?v=NwUijrm2U2w) by
    DVC on using GitHub Actions with R to automate data visualisation
    tasks.  

-   A useful [online resource](https://explainshell.com/) for explaining
    shell commands required to create components of the GitHub Actions
    YAML workflow.

-   <https://rpubs.com/glennwithtwons/reproducible-r-toolbox>

-   <https://goodresearch.dev/pipelines.html>