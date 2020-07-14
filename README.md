# FMT Source Documents
Source documents of FMT based on [mkdocs](https://www.mkdocs.org/).  To install *mkdocs*, refer to [page](https://www.mkdocs.org/#installation).

### Build And Deploy Documents
1. First checkout FMT GitHub Page repository:
```
git clone https://github.com/FirmamentPilot/docs.git
```
2. Checkout source document repository:
```
git clone https://github.com/FirmamentPilot/doc_src.git
```
3. Change folder to `docs/`, and run
```
mkdocs gh-deploy --config-file ../doc_src/mkdocs.yml --remote-branch master
```

If everything works fine, your documentation should be available at: https://FirmamentPilot.github.io/docs/

