# Custom PrairieLearn *external grader* image for CMPSC 5A/5B:
# the stock PrairieLearn Python grader, plus the `datascience` library.
#
# Used by questions whose info.json has
#   "externalGradingOptions": { "image": "ucsbcs/grader-python-datascience:latest",
#                               "entrypoint": "/python_autograder/run.sh" }
FROM prairielearn/grader-python:latest

# The datascience version is PINNED, and is the same across every image in
# this repo (and must match what the course repo vendors into
# serverFilesCourse/ for server.py). The single source of truth is
# datascience-version.txt at the root of this repo; the GitHub Actions
# workflows read it and pass it in as this build arg. The default below
# only matters for manual `docker build` runs without --build-arg.
# See README.md ("Why the version is pinned") for the rationale.
ARG DATASCIENCE_VERSION=0.18.1

RUN pip install --no-cache-dir "datascience==${DATASCIENCE_VERSION}"
