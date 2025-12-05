FROM ghcr.io/mamba-org/micromamba:2.4.0-ubuntu24.04

LABEL maintainer="James Gebbie-Rayet <james.gebbie@stfc.ac.uk>"

ARG AIIDAVERSION=2.6.1
ARG AMBERTOOLSVERSION=24
ARG PYTHONVERSION=3.11

# Install dependencies
RUN micromamba install -c conda-forge python=$PYTHONVERSION aiida-core=$AIIDAVERSION ambertools=$AMBERTOOLSVERSION && \
micromamba clean --all --yes
