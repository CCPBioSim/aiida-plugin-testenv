ARG BASE_IMAGE=ubuntu:24.04
FROM $BASE_IMAGE

LABEL maintainer="James Gebbie-Rayet <james.gebbie@stfc.ac.uk>"

# Define customisable args
ARG AIIDAVERSION=2.6.3
ARG GMXVERSION=2025.4
ARG PLUMEDVERSION=2.9.2
ARG PYTHONVERSION=3.11
ARG SYSTEM_USER=aiida
ARG SYSTEM_UID=1001
ARG SYSTEM_GID=121

# Fix: https://github.com/hadolint/hadolint/wiki/DL4006
# Fix: https://github.com/koalaman/shellcheck/wiki/SC3014
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

USER root

# Set OS level deps
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update --yes && \
    apt-get upgrade --yes && \
    apt-get install --yes --no-install-recommends \
    bison \
    build-essential \
    bzip2 \
    ca-certificates \
    cmake \
    curl \
    flex \
    erlang \
    gfortran \
    git \
    libbz2-dev \
    locales \
    openssh-client \
    sudo \
    vim \
    wget \
    xz-utils \
    zlib1g-dev && \
    apt-get clean && rm -rf /var/lib/apt/lists/* && \
    echo "en_US.UTF-8 UTF-8" > /etc/locale.gen && \
    locale-gen

# Configure env vars
ENV CONDA_DIR=/opt/conda \
    SHELL=/bin/bash \
    SYSTEM_USER="${SYSTEM_USER}" \
    SYSTEM_UID=${SYSTEM_UID} \
    SYSTEM_GID=${SYSTEM_GID} \
    LC_ALL=en_US.UTF-8 \
    LANG=en_US.UTF-8 \
    LANGUAGE=en_US.UTF-8
ENV PATH="${CONDA_DIR}/bin:${PATH}" \
    HOME="/home/${SYSTEM_USER}"

# Create SYSTEM_USER with name aiida  with UID=1001 and in the 'users' group
# and make sure these dirs are writable by the `users` group.
RUN echo "auth requisite pam_deny.so" >> /etc/pam.d/su && \
    sed -i.bak -e 's/^%admin/#%admin/' /etc/sudoers && \
    sed -i.bak -e 's/^%sudo/#%sudo/' /etc/sudoers && \
    addgroup --gid "${SYSTEM_GID}" "${SYSTEM_USER}" && \
    useradd -l -m -s /bin/bash -N --uid "${SYSTEM_UID}" --gid "${SYSTEM_GID}" "${SYSTEM_USER}" && \
    mkdir -p "${CONDA_DIR}" && \
    chown "${SYSTEM_USER}:${SYSTEM_GID}" "${CONDA_DIR}" "${HOME}" && \
    chmod g+w /etc/passwd

# make a github actions mount point.
RUN mkdir /github && \
    chown "${SYSTEM_USER}:${SYSTEM_GID}" /github

USER ${SYSTEM_UID}

# Download and install Micromamba, and initialize Conda prefix.
COPY --chown="${SYSTEM_UID}:${SYSTEM_GID}" gromacs/.condarc "${CONDA_DIR}/.condarc"
WORKDIR /tmp
RUN set -x && \
    arch=$(uname -m) && \
    if [ "${arch}" = "x86_64" ]; then \
        # Should be simpler, see <https://github.com/mamba-org/mamba/issues/1437>
        arch="64"; \
    fi && \
    wget --progress=dot:giga -O /tmp/micromamba.tar.bz2 \
        "https://micromamba.snakepit.net/api/micromamba/linux-${arch}/latest" && \
    tar -xvjf /tmp/micromamba.tar.bz2 --strip-components=1 bin/micromamba && \
    rm /tmp/micromamba.tar.bz2 && \
    ./micromamba install \
        --root-prefix="${CONDA_DIR}" \
        --prefix="${CONDA_DIR}" \
        --yes \
        'mamba' && \
    rm micromamba

# Add ~/.local/bin to PATH where the dependencies get installed via pip
# This require the package installed with `--user` flag in pip
ENV PATH=${PATH}:/home/${SYSTEM_USER}/.local/bin

# Install dependencies
RUN mamba install -c conda-forge mamba python=$PYTHONVERSION aiida-core=$AIIDAVERSION aiida-core.services=$AIIDAVERSION && \
    mamba clean --all --yes

# Change to root and build in /tmp.
USER root
WORKDIR /tmp

# Grab the specified version of plumed and untar the source.
RUN wget https://github.com/plumed/plumed2/archive/refs/tags/v${PLUMEDVERSION}.tar.gz && \
    tar xvf v${PLUMEDVERSION}.tar.gz && \
    rm v${PLUMEDVERSION}.tar.gz

# Configure, compile and install plumed.
WORKDIR /tmp/plumed2-${PLUMEDVERSION}
RUN ./configure --prefix=/opt/plumed2-${PLUMEDVERSION}
RUN make -j8
RUN make install

# Add plumed to the various paths.
ENV PATH=/opt/plumed2-${PLUMEDVERSION}/bin:$PATH
ENV LIBRARY_PATH=/opt/plumed2-${PLUMEDVERSION}/lib:$LIBRARY_PATH
ENV LD_LIBRARY_PATH=/opt/plumed2-${PLUMEDVERSION}/lib:$LD_LIBRARY_PATH
ENV PLUMED_KERNEL=/opt/plumed2-${PLUMEDVERSION}/lib/libplumedKernel.so
ENV PLUMED_VIMPATH=/opt/plumed2-${PLUMEDVERSION}/vim
ENV PYTHONPATH=/opt/plumed2-${PLUMEDVERSION}/python:$PYTHONPATH

# Back to tmp
WORKDIR /tmp

# Grab the specified version of gromacs and untar the source.
RUN wget ftp://ftp.gromacs.org/gromacs/gromacs-${GMXVERSION}.tar.gz && \
    tar xvf gromacs-${GMXVERSION}.tar.gz && \
    rm gromacs-${GMXVERSION}.tar.gz

# Patch gmx with plumed and make build dir
WORKDIR /tmp/gromacs-${GMXVERSION}
#RUN plumed patch -p -e gromacs-2025.4
RUN mkdir build

# Build gromacs without mpi and without cuda.
WORKDIR /tmp/gromacs-${GMXVERSION}/build
RUN cmake .. -DCMAKE_INSTALL_PREFIX=/opt/gromacs-${GMXVERSION} -DGMX_BUILD_OWN_FFTW=ON -DGMX_OPENMP=ON -DGMXAPI=OFF -DCMAKE_BUILD_TYPE=Release

RUN make -j8
RUN make install

# Cleanup and fix permissions.
RUN rm -r /tmp/gromacs-${GMXVERSION}
RUN chown ${SYSTEM_UID}:${SYSTEM_GID} -R /opt/gromacs-${GMXVERSION}

# Add the exec to the path.
ENV PATH=/opt/gromacs-${GMXVERSION}/bin:$PATH

# Switch back to USER aiida to avoid accidental container runs as root
USER ${SYSTEM_UID}
WORKDIR "${HOME}"
