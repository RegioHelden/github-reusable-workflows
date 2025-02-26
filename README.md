# github-reusable-workflows

## github-make-release.yaml

Downloads package artifacts from `python-build-package.yaml`, signs them and creates a new release on GitHub.

## python-build-package.yaml

Build Python packages with wheel and source tarball using pypa/build and store them as artifacts.

Parameters:
* `python-version` used during build process

## python-publish-pypi.yaml

Downloads package artifacts from `python-build-package.yaml` and publish them as a new release on PyPI.

## python-ruff.yaml

Lint and format using ruff.

Parameters:
* `ruff-version`
