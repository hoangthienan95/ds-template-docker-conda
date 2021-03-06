#from https://towardsdatascience.com/conda-pip-and-docker-ftw-d64fe638dc45

FROM ubuntu:18.04

LABEL maintainer="pughdr <david.pugh@kaust.edu.sa>"

SHELL [ "/bin/bash", "--login", "-c" ]

RUN apt-get update --fix-missing && \
    apt-get install -y wget bzip2 curl git && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Create a non-root user
ARG username=replace-user-here
ARG uid=1000
ARG gid=100
ENV USER $username
ENV UID $uid
ENV GID $gid
ENV HOME /home/$USER

RUN adduser --disabled-password \
    --gecos "Non-root user" \
    --uid $UID \
    --gid $GID \
    --home $HOME \
    $USER

#Newer versions of Docker support copying files as a non-root user,
# however the version of Docker available on DockerHub does not yet support copying as a non-root user
# so if you want to set up automated builds for your Git repositories you will need to copy everything as root.
COPY environment.yml environment-extras.yml requirements.txt /tmp/
RUN chown $UID:$GID /tmp/environment.yml /tmp/environment-extras.yml /tmp/requirements.txt

COPY postBuild /usr/local/bin/postBuild.sh
RUN chown $UID:$GID /usr/local/bin/postBuild.sh && \
    chmod u+x /usr/local/bin/postBuild.sh

COPY docker/entrypoint.sh /usr/local/bin/
RUN chown $UID:$GID /usr/local/bin/entrypoint.sh && \
    chmod u+x /usr/local/bin/entrypoint.sh

#After copying over the config files as root, I switch over to the non-root user
USER $USER

# install miniconda
ENV MINICONDA_VERSION latest
ENV CONDA_DIR $HOME/miniconda3
RUN wget --quiet https://repo.anaconda.com/miniconda/Miniconda3-$MINICONDA_VERSION-Linux-x86_64.sh -O ~/miniconda.sh && \
    chmod +x ~/miniconda.sh && \
    ~/miniconda.sh -b -p $CONDA_DIR && \
    rm ~/miniconda.sh

# make non-activate conda commands available
ENV PATH=$CONDA_DIR/bin:$PATH

# make conda activate command available from /bin/bash --login shells
RUN echo ". $CONDA_DIR/etc/profile.d/conda.sh" >> ~/.profile

# make conda activate command available from /bin/bash --interative shells
RUN conda init bash

# create a project directory inside user home
ENV PROJECT_DIR $HOME/app
RUN mkdir $PROJECT_DIR
WORKDIR $PROJECT_DIR

# build the conda environment
ENV ENV_PREFIX $PROJECT_DIR/env
RUN conda update --name base --channel defaults conda && \
    conda install mamba -c conda-forge &&\
    mamba env create --prefix $ENV_PREFIX --file /tmp/environment.yml --force && \
    conda clean --all --yes

# run the postBuild script to install the JupyterLab extensions
RUN conda activate $ENV_PREFIX && \
    #while read extra_pkgs; do conda install -c conda-forge -c defaults --yes $extra_pkgs; done < /tmp/environment-extras.yml &&\
    /usr/local/bin/postBuild.sh && \
    conda deactivate

# use an entrypoint script to insure conda environment is properly activated at runtime
ENTRYPOINT [ "/usr/local/bin/entrypoint.sh" ]

# default command will be to launch JupyterLab server for development
CMD [ "jupyter", "lab", "--no-browser", "--ip", "0.0.0.0" ]

# build with the following command:

# docker image build \
#   --build-arg username=$USER \
#   --build-arg uid=$UID \
#   --build-arg gid=$GID \
#   --file Dockerfile \
#   --tag $IMAGE_NAME:$IMAGE_TAG \
#   ../

#OR, alternatively,
#docker-compose up --build
