# Applied Deep Learning
Website for applied deep learning, using our publicly available annotation/conversion tools and docker images.

https://www.data-mining.co.nz/applied-deep-learning/

## Local

### Installation

```bash
python3 -m venv venv
./venv/bin/pip install mkdocs==1.4.2 mkdocs-video==1.3.0 jinja2==3.1.2 "Markdown<3.4.0" mkdocs-table-reader-plugin==1.1.0 mkdocs-material==8.5.10
```

### Serving

```bash
./venv/bin/mkdocs serve
```

## Deploying

Any push will trigger a rebuild of the site on github via github actions:

[.github/workflows/main.yml](.github/workflows/main.yml)
