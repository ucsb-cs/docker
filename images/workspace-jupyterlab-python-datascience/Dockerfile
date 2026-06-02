FROM prairielearn/workspace-jupyterlab-python:latest

USER root

# 1. Set the Matplotlib configuration directory to a standard writable path
ENV MPLCONFIGDIR=/tmp/matplotlib-cache

# 2. Pre-create the directory and open its permissions so any workspace user can read/write to it
RUN mkdir -p /tmp/matplotlib-cache && chmod -R 777 /tmp/matplotlib-cache

# 3. Install the datascience package
RUN /opt/conda/bin/pip install --no-cache-dir datascience
