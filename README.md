# mkdocs-k8s

Documentation site with summarized knowledge and axperience on k8s.
The site is developed using [mkdocs](mkdocs.org) site generator.

The site is deployed to the gh-pages branch of this repository using mkdocs sw.

## mkdocs Project Layout
```
mkdocs.yml    # The configuration file.
docs/
index.md  # The documentation homepage.
...       # Other markdown pages, images and other files.
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
