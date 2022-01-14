# mkdocs-k8s

This is a documentation site with summarized knowledge and experience on k8s.
The site is developed using [mkdocs](mkdocs.org) site generator and deployed to the gh-pages branch of this repository using `mkdocs` sw.

## Project Directory Structure
```
mkdocs.yml    # mkdocs configuration file.
docs/
  index.md        # The documentation homepage.
  img/
    favicon.ico   # image to be used by mkdocs as site favicon
    ...           # other images referenced from the documentation files
  ...             # Other documentation files
```

## Change/Update Instructions

1. Clone the repository
2. Introduce changes to the `main` branch
3. Check the changes locally with `mkdocs serve`
4. Build the site locally with `mkdocs build`
5. Deploy the site with `mkdocs gh-deploy`

---

## mkdocs Command Reference

* `mkdocs new [dir-name]` - Create a new project.
* `mkdocs serve` - Start the live-reloading docs server.
* `mkdocs build` - Build the documentation site.
* `mkdocs -h` - Print help message and exit.

---
