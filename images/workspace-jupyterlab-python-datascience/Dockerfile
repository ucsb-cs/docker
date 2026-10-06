# Custom PrairieLearn *workspace* image for CMPSC 5A/5B:
# the stock PrairieLearn JupyterLab workspace, plus the `datascience` library.
#
# Used by questions whose info.json has
#   "workspaceOptions": { "image": "ucsbcs/workspace-jupyterlab-python-datascience:latest", ... }
#
# NOTE: this image has NO autograder in it. Do not use it as an
# externalGradingOptions image; use grader-python-datascience for that.
FROM prairielearn/workspace-jupyterlab-python:latest

USER root

# 1. Set the Matplotlib configuration directory to a standard writable path
ENV MPLCONFIGDIR=/tmp/matplotlib-cache

# 2. Pre-create the directory and open its permissions so any workspace user can read/write to it
RUN mkdir -p /tmp/matplotlib-cache && chmod -R 777 /tmp/matplotlib-cache

# 3. Install the datascience package.
# The version is PINNED and shared across all images in this repo; the
# single source of truth is datascience-version.txt at the repo root, which
# the GitHub Actions workflows pass in as this build arg. The default below
# only matters for manual `docker build` runs without --build-arg.
# See README.md ("Why the version is pinned") for the rationale.
ARG DATASCIENCE_VERSION=0.18.1

RUN /opt/conda/bin/pip install --no-cache-dir "datascience==${DATASCIENCE_VERSION}"
