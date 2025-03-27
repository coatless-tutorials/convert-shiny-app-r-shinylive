# Deploying an R Shinylive App via GitHub Pages through GitHub Actions

This repository demonstrates how to deploy an R Shinylive app using GitHub 
Actions and GitHub Pages.

> [!NOTE]
>
> Please note that this guide is **not** affiliated with Posit, Shiny,
> or Shinylive project. It is a community-driven tutorial to help you get started
> using Shinylive with R Shiny Applications. 

### Other Guides

- [Creating an R Shinylive App inside a Quarto Document](https://github.com/coatless-quarto/r-shinylive-demo)
- **Deploying an R Shinylive App via GitHub Pages through GitHub Actions**
- [Data Inclusion in a Shinylive App](https://tutorials.thecoatlessprofessor.com/r-shinylive-data-include/)
- [Deploying an R Shiny Dashboard App through R Shinylive](https://github.com/coatless-dashboard/r-shinylive-dashboard-app)
- [Deploying a Python Shinylive App via GitHub Pages through GitHub Actions](https://github.com/coatless-tutorials/convert-py-shiny-app-to-py-shinylive-app)

### Updates

- **03/26/2025**
  - We've fixed the version of the `shinylive` R package to v0.3.0 and 
    updated the version of GitHub Actions to either v3 or v4 to avoid deprecation
    warnings.
- **03/22/2024**
  - Initial release of the tutorial.

## Background

The initial app source code is from a [StackOverflow Question](https://stackoverflow.com/questions/78160039/using-shinylive-to-allow-deployment-of-r-shiny-apps-from-a-static-webserver-yiel) by [Faustin Gashakamba](https://stackoverflow.com/users/5618354/faustin-gashakamba)

## Shinylive App Size

Shinylive apps require around 60 MB or more data due to their dependency on the initial webR base and all {shiny} package dependencies. When moving away from a compute server running Shiny Server to a generic web server, the trade-off lies in the amount of user bandwidth required. So, the users internet and computer is paying for the ability to view the app compared to the old status quo where the host paid a license fee (to Posit), server and maintenance costs.

You can inspect the current list of package dependencies for a basic Shiny app using the following R code:

```r
pkg_db <- tools::CRAN_package_db()
shiny_pkg_dependencies <- tools::package_dependencies(
  packages = c("shiny"),
  recursive = TRUE,
  db = pkg_db
)

shiny_pkg_dependencies
#> $shiny
#>  [1] "methods"     "utils"       "grDevices"   "httpuv"      "mime"       
#>  [6] "jsonlite"    "xtable"      "fontawesome" "htmltools"   "R6"         
#> [11] "sourcetools" "later"       "promises"    "tools"       "crayon"     
#> [16] "rlang"       "fastmap"     "withr"       "commonmark"  "glue"       
#> [21] "bslib"       "cachem"      "ellipsis"    "lifecycle"   "base64enc"  
#> [26] "jquerylib"   "memoise"     "sass"        "digest"      "Rcpp"       
#> [31] "cli"         "magrittr"    "stats"       "graphics"    "fs"         
#> [36] "rappdirs"
```

Adding more R packages through in-app dependencies will expand this list and increase the download size of each app. 


## Deploying Automatically with GitHub Actions

### Serving a Website from a GitHub Repository

One of the standard approaches when working with web deployment is to use the [`gh-pages` branch deployment technique](https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-from-a-branch). However, this approach is problematic for sharing the app on GitHub pages since you can quickly bloat the size of the repository with binary data, which impacts being able to quickly clone the repository and decreases the working efficiency.

Instead, opt for the [GitHub Actions approach](https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow), which is preferred because it doesn't store artifacts from converting a Shiny App into a Shinylive App inside the repository. Plus, this approach allows for Shinylive apps to be deployed up to the [GitHub Pages maximum of about 1 GB](https://docs.github.com/en/pages/getting-started-with-github-pages/about-github-pages#usage-limits). 

### Project Layout

For this to work, we're advocating for a repository structure of: 

```sh
.
├── .github
│   └── workflows
│       └── build-and-deploy-shinylive-r-app.yml
├── README.md
├── DESCRIPTION  
└── app.R
```

The source for the R Shiny app can be placed into the [`app.R`](app.R) file,
the DESCRIPTION file can be used to specify the dependencies of the app,
and, then, use the following GitHub Action in `.github/workflows/build-and-deploy-shinylive-r-app.yml` to build and deploy the shinylive app every time the repository is updated.

## DESCRIPTION File

The DESCRIPTION file is used to specify the dependencies of the app. For the
example app, the DESCRIPTION file is as follows:

```default
Package: shinyliveconvertdemo
Title: Demo converting an R Shiny application to Shinylive
Version: 0.0.1
Description: List the package dependencies here for the GitHub action to automatically
  include the packages on the build. This is required for r-shinylive version >= 0.3.0.
Imports: 
    shiny,
    shinythemes
Encoding: UTF-8
```


## GitHub Action Workflow for Converting and Deploying

The following workflow contains a single step that encompasses both the build and deploy phases. For more details about customizing the conversion step or the deployment step, please see the two notes that immediately follow from the workflow.

```yaml
on:
    push:
      branches: [main, master]
    release:
        types: [published]
    workflow_dispatch: {}
   
name: demo-r-shinylive-app

jobs:
    demo-website:
      runs-on: ubuntu-latest
      # Only restrict concurrency for non-PR jobs
      concurrency:
        group: r-shinylive-website-${{ github.event_name != 'pull_request' || github.run_id }}
      # Describe the permissions for obtain repository contents and 
      # deploying a GitHub pages website for the repository
      permissions:
        contents: read
        pages: write
        id-token: write
      steps:
        # Obtain the contents of the repository
        - name: "Check out repository"
          uses: actions/checkout@v4

        # Install R on the GitHub Actions worker
        - name: "Setup R"
          uses: r-lib/actions/setup-r@v2
  
        # Install dependency and pin the shinylive R package dependency
        # to the current version on CRAN
        - name: "Setup R dependency for Shinylive App export"
          uses: r-lib/actions/setup-r-dependencies@v2
          with:
            ## Install required dependencies from the DESCRIPTION file
            ## Required for shinylive >= 0.3.0
            packages: |
              cran::shinylive@0.3.0 
              local::.
  
        # Export the current working directory as the shiny app
        # using the pinned version of the Shinylive R package
        - name: Create R Shinylive App from working directory files
          shell: Rscript {0}
          run: |
           shinylive::export(".", "_site")

        # Upload a tar file that will work with GitHub Pages
        # Make sure to set a retention day to avoid running into a cap
        # This artifact shouldn't be required after deployment onto pages was a success.
        - name: Upload Pages artifact
          uses: actions/upload-pages-artifact@v3
          with: 
            retention-days: 1
        
        # Use an Action deploy to push the artifact onto GitHub Pages
        # This requires the `Action` tab being structured to allow for deployment
        # instead of using `docs/` or the `gh-pages` branch of the repository
        - name: Deploy to GitHub Pages
          id: deployment
          uses: actions/deploy-pages@v4
```

> [!NOTE]
>
> We have pinned the version of shinylive package on CRAN to v0.3.0 to ensure 
> consistency across deployments. If you want to use the latest version, please
> remove the `## Pin version to ensure consistency` line from the
> `Setup R dependency for Shinylive App export` step.

> [!NOTE]
>
> GitHub Actions may require the `actions/upload-pages-artifact@v3` and 
> `actions/deploy-pages@v4` actions to be updated to the later versions if
> you encounter GitHub Action runner deprecation warnings.


#### Conversion Assumptions

When exporting the R Shiny app, we assume:

1. The app is located in the working directory, denoted by `.`.
2. The deployment folder or what should be shown on the web is `_site`.

If these assumptions don't align with your setup, please modify the conversion step accordingly.

```yaml
  - name: Create R Shinylive App from working directory files
    shell: Rscript {0}
    run: |
      shinylive::export(".", "_site")
```

#### Customize Deployment Path

The output directory `_site` for the converted Shinylive app is used as it's the default path for the [`upload-pages-artifact`](https://github.com/actions/upload-pages-artifact) action. You can change this by supplying a `path` parameter under `with` in the "Upload Pages artifact" step, e.g.

```yaml
- name: Upload Pages artifact
  uses: actions/upload-pages-artifact@v3
  with: 
    retention-days: 1
    path: "new-path-here"
```

## Enabling GitHub Pages Deployment

To enable deployment through GitHub Actions to GitHub Pages, please enable it on the repository by:

- Clicking on the repository's **Settings** page
- Selecting **Pages** on the left sidebar.
- Picking the **GitHub Actions** option in the **Source** drop-down under the Build and Deployment section.
- Ensuring that **Enforced HTTPS** is checked. 

[![Example annotation of the repository's Settings page for GitHub Actions deployment][1]][1]

## Working Example

You can view the example shinylive-ified [app.R](app.R) source included in the repository here:

<https://tutorials.thecoatlessprofessor.com/convert-shiny-app-r-shinylive>

Keep in mind that the exported app size is not mobile-friendly (approximately 66.8 MB).

If you are data constrained, you can see the app in this screenshot: 

[![Example of the working shinylive app][2]][2]

Moreover, you can view one of the deployments app's commit size here: 

[![Summary of the deployment of the shinylive app][3]][3]

# Fin

Huzzah! You've successfully learned how to deploy an R Shinylive app via GitHub Pages using GitHub Actions.
Now, unleash your creativity and start building!

  [1]: https://github.com/coatless-tutorials/convert-shiny-app-r-shinylive/assets/833642/b9fff36a-26c6-4b8e-bb6c-ea4b3d6fe811
  [2]: https://github.com/coatless-tutorials/convert-shiny-app-r-shinylive/assets/833642/245ed1f4-9343-4d30-b395-a954c8d85f08
  [3]: https://github.com/coatless-tutorials/convert-shiny-app-r-shinylive/assets/833642/bdd270b7-c4ae-4669-bae7-9abc6b4edce7
