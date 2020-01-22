================
hubploy-template
================

Template for fully specified JupyterHub deployment with hubploy.

Step 0: Set up your pre-requisites
==================================

hubploy does *not* manage your cloud resources - only your *Kubernetes*
resources. You should use some other means to create your cloud
resources. At a minimum, hubploy expects a Kubernetes cluster with [helm
installed](https://zero-to-jupyterhub.readthedocs.io/en/latest/setup-jupyterhub/setup-helm.html).
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
something more descriptive. ``git commit`` the result as well.

.. code:: bash

   git mv deployments/myhub deployments/<your-hub-name>
   git commit

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

#. Customize the hub image. `repo2docker`_ is used to build the image,
   so you can put any of the `supported configuration files`_ under
   ``deployments/<hub-image>/image``. You *must* make a git commit after
   modifying this for
   ``hubploy build <hub-name> --push --check-registry`` to work, since
   it uses the commit hash as the image tag.

#. Customize hub configuration with various YAML files.

   #. ``hub/values.yaml`` is common to *all* hubs that exist in this repo
      (multiple hubs can live under ``deployments/``).

   #. ``deployments/<hub-name>/config/common.yaml`` is where most of the config specific
      to each hub should go. Examples include memory / cpu limits, home directory
      definitions, etc

   #. ``deployments/<hub-name>/config/staging.yaml`` and ``deployments/<hub-name>/config/prod.yaml``
      are files specific to the staging & prod versions of the hub. These should be
      *as minimal as possible*. Ideally, only DNS entries, IP addresses, should be here.

   #. ``deployments/<hub-name>/secrets/staging.yaml`` and ``deployments/<hub-name>/secrets/prod.yaml``
       should contain information that mustn't be public. This would be proxy / hub
       secret tokens, any authentication tokens you have, etc. These files *must* be
       protected by something like `git-crypt <https://github.com/AGWA/git-crypt>`_ or
       `sops <https://github.com/mozilla/sops`_. **THIS REPO TEMPLATE DOES NOT HAVE
       THIS PROTECTION SET UP YET**


You can customize the staging hub, deploy it with ``hubploy deploy <hub-name> hub staging``, and iterate until you like how it behaves.

Step 7: Deploy to prod
======================

You can then do a production deployment with: ``hubploy deploy <hub-name> hub prod``, and
test it out!

Step 8: Setup git-crypt for secrets
===================================

`git-crypt <https://github.com/AGWA/git-crypt>`_ is used to keep encrypted secrets in the
git repository. We would eventually like to use something like `sops <https://github.com/mozilla/sops>`_
but for now...

1. Install git-crypt. You can get it from brew or your package manager.

2. In your repo, initialize it.

   .. code:: bash

      git crypt init

3. In ``.gitattributes`` have the following contents:

   .. code::

      deployments/*/secrets/** filter=git-crypt diff=git-crypt
      deployments/**/secrets/** filter=git-crypt diff=git-crypt
      support/secrets.yaml filter=git-crypt diff=git-crypt

4. Make a copy of your encryption key. This will be used to decrypt the secrets.
   You will need to share it with your CD provider, and anyone else.

   .. code::

      git crypt export-key key

   This puts the key in a file called 'key'

Step 9: GitHub workflows
========================

1. Get a base64 copy of your key

   .. code:: block

      cat key | base64

2. Put it as a secret named GIT_CRYPT_KEY in github secrets.

3. Make sure you change the `myhub` to your deployment name in the
   workflows under `.github/workflows`.

4. Push to the staging branch, and check out GitHub actions, to
   see if your action goes to completion.

5. If the staging action succeeds, make a PR from staging to prod,
   and merge this PR. This should also trigger an action - see if
   this works out.

**Note**: *Always* make a PR from staging to prod, never push directly to
prod. We want to keep the staging and prod branches as close to each
other as possible, and this is the only long term guaranteed way to do
that.


TODO
====

1. What kinda kubernetes setup this needs

.. _repo2docker: https://repo2docker.readthedocs.io/
.. _supported configuration files: https://repo2docker.readthedocs.io/en/latest/config_files.html
