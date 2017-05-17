``hotfix-ubuntu``
=================

A tool to package up hotfixes for Red Hat Ceph Storage for Ubuntu.

This tool uses `rhcephpkg <https://github.com/red-hat-storage/rhcephpkg>`_ to
download hotfix builds from Red Hat's build storage system (`chacra
<https://github.com/ceph/chacra>`_), generate a local apt repository with those
builds (using `reprepro <http://mirrorer.alioth.debian.org/>`_), and assemble
everything into a single tarball for customers.

Example
=======

(prerequisite: Install and configure rhcephpkg and reprepro)

Clone from Git and run it::

  git clone https://github.com/ktdreyer/hotfix-ubuntu
  cd hotfix-ubuntu
  ./hotfix-ubuntu <bug-id> <build-nvr>

This will create a ``hotfix-bzXXXX.tar.gz`` in your local working directory.

Patch handling process
======================

This guide walks through the hotfix patch handling process for Ubuntu.

#. Determine the downstream (``ceph-2-rhel-patches``) branch point at which
   the hotfix should be created.

   Let's say you're starting from something very general like a release, for
   example "RH Ceph Storage 2.2".

   #. Look up the ceph-2.2 QuarterlyUpdate release in the Errata Tool to find
      the advisory that shipped for that, for example
      https://errata.devel.redhat.com/advisory/26175

   #. Take note of the advisory ID number, ``26175``.

   #. Look up the rhcs-metadata.git ``builds-*.txt`` list that corresponds to
      that advisory::

        ls builds-*26175*.txt

   #. Determine the ceph NVR from that ``builds-ceph-2.2-26175-xenial.txt``
      list.

   #. Look back through dist-git's log for that NVR. This is the dist-git
      commit point to branch from.
      ::

        git log origin/ceph-2-ubuntu --grep 10.2.5-28redhat1

   #. Check out this dist-git point::

        git checkout 16ab8afc351af645e716fd7d40cab82319067744

   #. Determine the ``patch-queue/`` branch point::

        grep COMMIT debian/rules

      This will give you a sha1 like ``export
      COMMIT=033f137cde8573cfc5a4662b4ed6a63b8a8d1464``

#. Create the hotfix branch, so the developer can cherry-pick here::

     git checkout -b ceph-2.2-rhel-patches-hotfix-bz1445891 033f137cde8573cfc5a4662b4ed6a63b8a8d1464
     git push -u patches ceph-2.2-rhel-patches-hotfix-bz1445891

#. The developer cherry-picks to the hotfix branch. For example, on
   their computer, they might use `downstream-cherry-picker
   <https://github.com/ktdreyer/downstream-cherry-picker>`_::

     git fetch patches
     git checkout ceph-2.2-rhel-patches-hotfix-bz1445891
     downstream-cherry-picker https://github.com/... 1445891
     git push patches ceph-2.2-rhel-patches-hotfix-bz1445891

#. Back on our computer, create our private Ubuntu dist-git branch::

     git branch ceph-2.2-ubuntu-hotfix-bz1445891 ceph-2.2-ubuntu

#. Create our corresponding private ``patch-queue/`` branch::

     git branch patch-queue/ceph-2.2-ubuntu-hotfix-bz1445891 ceph-2.2-rhel-patches-hotfix-bz1445891

#. Switch to the ubuntu dist-git branch::

     git checkout ceph-2.2-ubuntu-hotfix-bz1445891

#. Write out the new ``.patch`` file series, bump ``changelog``,
   update ``rules``, etc::

     rhcephpkg patch

#. The ``rhcephpkg patch`` command will have bumped the Version-Release field
   from ``10.2.5-28redhat1`` to ``10.2.5-29redhat1``. Edit
   ``debian/changelog`` to manually set the Version-Release to a hotfix
   value instead.

   Also add the "dist" string suffix here (``xenial``), since this is going
   to be a ``localbuild`` rather than in Jenkins ... although eventually I want
   to get rid of this part, and make Jenkins handle this properly.
   ::

     10.2.5-28.1.bz1445891redhat1xenial

#. Since we're on a non-standard dist-git branch, we also need to edit
   ``debian/gbp.conf``. Open ``debian/gbp.conf`` in your editor, and set::

     debian-branch = ceph-2.2-ubuntu-hotfix-bz1445891

#. Amend the dist-git commit to include your updates to ``debian/changelog``
   and ``debian/gbp.conf``::

     git commit -a --amend

#. Do a local build::

     rhcephpkg localbuild --dist xenial
