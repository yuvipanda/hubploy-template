================
hubploy-template
================

Template for fully specified JupyterHub deployment with hubploy.

Step 0: Set up your pre-requisites
==================================

hubploy does *not* manage your cloud resources - only your *Kubernetes*
resources. You should use some other means to create your cloud
resources. At a minimum, hubploy expects a Kubernetes cluster with [helm
installed]
(https://zero-to-jupyterhub.readthedocs.io/en/latest/setup-jupyterhub/setup-helm.html).
Many installations want to use a shared file system for home
directories, so in those cases you want to have that managed outside
hubploy as well.

You also need the following tools installed:

#. Your cloud vendor's commandline tool.

   #. `Google Cloud SDK <https://cloud.google.com/sdk/>`_ for Google Cloud
   #. `AWS CLI <https://aws.amazon.com/cli/>`_ for AWS
   #. `Azure CLI <https://docs.microsoft.com/en-us/cli/azure/>`_ for Azure

#. A local install of `helm 2 <https://helm.sh/>`_. Note that helm 3 is *not*
   supported yet. The client version should match the version on your server (you
   can find your server version with ``helm version``.

#. A `docker environment <https://docs.docker.com/install/>`_ that you can use. This
   is only needed when building images.

Step 1: Install hubploy
=======================

.. code:: bash

   python3 -m venv .
   source bin/activate
   python3 -m pip install -r requirements.txt

This installs hubploy its dependencies

Step 2: Rename the hub to something more sensible
=================================================

Each directory inside ``deployments/`` represents an installation of
JupyterHub. The default is called ``myhub``, but *please* rename it to
something more descriptive

Step 3: Fill in all the config details
======================================

You need to find all things marked TODO and fill them in. In particular,

1. ``hubploy.yaml`` needs information about where your docker registry &
   kubernetes cluster is, and paths to access keys as well.
2. ``secrets/prod.yaml`` and ``secrets/staging.yaml`` require secure
   random keys you can generate and fill in.

Step 4: Build and push the image
================================

Run:

.. code:: bash

   hubploy build <hub-name> --push --check-registry

This should check if the user image for your hub needs to be rebuilt,
and if so, it’ll build and push it.

Step 5: Deploy the staging hub
==============================

Each hub will always have two versions - a *staging* hub that isn’t used
by actual users, and a *production* hub that is. These two should be
kept as similar as possible, so you can fearlessly test stuff on the
staging hub without feaer that it is going to crash & burn when deployed
to production.

To deploy to the staging hub,

.. code:: bash

   hubploy deploy <hub-name> hub staging

This should take a while, but eventually return successfully. You can
then find the public IP of your hub with:

.. code:: bash

   kubectl -n <hub-name>-staging get svc public-proxy

If you access that, you should be able to get in with any username &
password.

The defaults provision each user their own EBS / Persistent Disk, so
this can get expensive quickly :) Watch out!

Step 6: Customize your hub
==========================

You can now customize your hub in two major ways:

1. Customize the hub image. `repo2docker`_ is used to build the image,
   so you can put any of the `supported configuration files`_ under
   ``deployments/<hub-image>/image``. You *must* make a git commit after
   modifying this for
   ``hubploy build <hub-name> --push --check-registry`` to work, since
   it uses the commit hash as the image tag.

2. Customize hub configuration with various YAML files.

   a. ``hub/values.yaml`` is common to *all* hubs that exist in this
      repo (multiple hubs

.. _repo2docker: https://repo2docker.readthedocs.io/
.. _supported configuration files: https://repo2docker.readthedocs.io/en/latest/config_files.html
