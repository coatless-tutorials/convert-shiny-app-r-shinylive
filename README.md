# Deploying an R Shinylive App via GitHub Pages through GitHub Actions

This repository demonstrates how to deploy an R Shinylive app using GitHub Actions and GitHub Pages.

The initial app source code is from a [StackOverflow Question](https://stackoverflow.com/questions/78160039/using-shinylive-to-allow-deployment-of-r-shiny-apps-from-a-static-webserver-yiel) by [Faustin Gashakamba](https://stackoverflow.com/users/5618354/faustin-gashakamba).

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

Please **avoid** using the [`gh-pages` branch deployment technique](https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-from-a-branch) for sharing the app on GitHub pages. 

Instead, opt for the [GitHub Actions approach](https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow), which is preferred because it doesn't store artifacts from converting a Shiny App into a Shinylive App inside the repository. Plus, this approach allows for Shinylive apps to be deployed up to the [GitHub Pages maximum of about 1 GB](https://docs.github.com/en/pages/getting-started-with-github-pages/about-github-pages#usage-limits). 

Specifically, we're advocating for a file structure of: 

```sh
.
├── .github
│   └── workflows
│       └── build-and-deploy-shinylive-r-app.yml
├── README.md
└── app.R
```

Thus, we could place the example app source code into `app.R` and, then, use the following GitHub Action in `.github/workflows/build-and-deploy-shinylive-r-app.yml` to build and deploy the shinylive app every time the repository is updated:

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
  
        # Install and pin the shinylive R package dependency
        - name: "Setup R dependency for Shinylive App export"
          uses: r-lib/actions/setup-r-dependencies@v2
          with:
            packages:
              cran::shinylive@0.1.1
  
        # Export the current working directory as the shiny app
        # using the pinned version of the Shinylive R package
        - name: Create Shinylive App from working directory files
          shell: Rscript {0}
          run: |
           shinylive::export(".", "_site")

        # Upload a tar file that will work with GitHub Pages
        # Make sure to set a retention day to avoid running into a cap
        # This artifact shouldn't be required after deployment onto pages was a success.
        - name: Upload Pages artifact
          uses: actions/upload-pages-artifact@v2
          with: 
            retention-days: 1
        
        # Use an Action deploy to push the artifact onto GitHub Pages
        # This requires the `Action` tab being structured to allow for deployment
        # instead of using `docs/` or the `gh-pages` branch of the repository
        - name: Deploy to GitHub Pages
          id: deployment
          uses: actions/deploy-pages@v2
```

Deployment with this approach requires it to be enabled by:

- Clicking on the repository's **Settings** page
- Selecting **Pages** on the left sidebar.
- Picking the **GitHub Actions** option in the **Source** drop-down under the Build and Deployment section.
- Ensuring that **Enforced HTTPS** is checked. 

[![Example annotation of the repository's Settings page for GitHub Actions deployment][1]][1]


## Working Example

For a comprehensive example demonstrating deployment, documentation, and a functional version of your app, refer to the following:

- Repository: https://github.com/coatless-tutorials/convert-shiny-app-r-shinylive
- Deployed shiny app: https://tutorials.thecoatlessprofessor.com/convert-shiny-app-r-shinylive/

Some quick screenshots that describe whats up:

[![Example of the working shinylive app][2]][2]

[![Summary of the deployment of the shinylive app][3]][3]


  [1]: https://i.stack.imgur.com/kF2pf.jpg
  [2]: https://i.stack.imgur.com/DQ3Z1.jpg
  [3]: https://i.stack.imgur.com/Gzzy8.jpg