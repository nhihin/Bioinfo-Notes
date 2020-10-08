# R/RStudio

## Setting up WorkflowR

1. Make a new project on RStudio by going `File >> New Project >> New Directory`.
2. Install [workflowR](https://jdblischak.github.io/workflowr/articles/wflow-01-getting-started.html) and load the package using `library(workflowr)`. 
3. Run the following in the Console to set up WorkflowR:

```{r}
wflow_start("~/Projects/PROJECTNAME", existing = TRUE, overwrite = TRUE)
```

4. Run `wflow_build()` in the Console. 
5. Build and commit the website using `wflow_publish()`. 

**Note**: The status of files can be checked using `wflow_status()`.

6. To deploy the website, run the following in Console: `wflow_use_github("nhihin")` (replace with GitHub username) and select option 1 to automatically create the repository. 
7. The Git tab on RStudio can now be used as normal to commit, push, and pull files. If it isn't showing up, usually restarting RStudio fixes this.

### Making a new analysis file

- Use `wflow_open("analysis/your-file-name.Rmd")` to make a new analysis RMarkdown. 
- Files must be built using `wflow_build("analysis/your-file-name.Rmd")` and committed using `wflow_publish("analysis/your-file-name.Rmd")`. 
- Alternatively, if `wflow_publish` gives an [error](https://github.com/jdblischak/workflowr/issues/188), just build first using `flow_build` and commit the file(s) using the Git tab on RStudio. Not sure if this is bad practice but it seems to work fine. (Or try to add a commit message - for some reason this works!)

### Editing the navigation bar

- This is important to do early on, as the navigation bar defined will be used across every analysis file created. This saves time if defined early on. 

1. Open the file in `analysis/_site.yml`. 
2. Under `left:` is where you can define links in the navigation bar. `Text` is the name of the link while `href` is the URL of the link. This is a relative url pointing to the `analysis` directory so no need to put in the full path.
3. Typically one link for one analysis file for each specific step of the analysis. The `menu:` thing is used to define a drop-down menu, which is helpful for organising many analysis pages which are related to each other. 

Example:

```
name: Project Name
output_dir: ../docs
navbar:
  title: Project Name
  left:
  - text: Home
    href: index.html
  - text: About
    href: about.html
  - text: Setup
    href: setup.html
    menu:
      - text: "Pre-processing"
      - text: "cellranger pipeline"
        href: cellranger.html
      - text: "Initial analysis"
      - text: "Merge datasets with Seurat"
        href: import-data.html
      - text: "Aggregate with cellranger"
        href: import-merged-data.html
  - text: License
    href: license.html
  right:
  - icon: fa-github
    text: Source code
    href: https://github.com/nhihin/Heart-scRNAseq
output:
  workflowr::wflow_html:
    toc: yes
    toc_float: yes
    theme: cosmo
    highlight: textmate
```

## Git

### If the Push/Pull buttons are greyed out on RStudio

- Go to the Terminal, navigate into the project directory, and use `git push -u origin master`. 

# Conda

Links:

- [Conda cheatsheet](https://docs.conda.io/projects/conda/en/4.6.0/_downloads/52a95608c49671267e40c689e0bc00ca/conda-cheatsheet.pdf)
- [Setup Conda on HPC](https://bitbucket.org/sahmri_bioinformatics/pipeline-resources/wiki/userSetUp)
- [Install Conda packages offline](https://docs.conda.io/projects/conda/en/latest/user-guide/concepts/installing-with-conda.html)
