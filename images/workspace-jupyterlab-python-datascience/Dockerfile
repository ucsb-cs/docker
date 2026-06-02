# Use the original PrairieLearn JupyterLab image as the base
FROM prairielearn/workspace-jupyterlab-python:latest

# Required to bypass the non-root 'ubuntu' user restrictions
USER root

# This path is globally owned by root, requiring the elevation above
RUN /opt/conda/bin/pip install --no-cache-dir datascience