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
   ::

     10.2.5-28.1.bz1445891redhat1xenial

#. Since we're on a non-standard dist-git branch, we also need to edit
   ``debian/gbp.conf``. Open ``debian/gbp.conf`` in your editor, and set::

     debian-branch = ceph-2.2-ubuntu-hotfix-bz1445891

#. Amend the dist-git commit to include your updates to ``debian/changelog``
   and ``debian/gbp.conf``::

     git commit -a --amend

#. Push your new hotfix change to the remote::

     git push origin ceph-2.2-ubuntu-hotfix-bz1445891

#. Do a build in Jenkins::

     rhcephpkg build

At this point you can use the ``hotfix-ubuntu`` script in this repository in
order to generate a small tarball that contains the fix, or else you can
generate a full compose (pros: looks similar to the released product. supports
multi-distros. cons: more effort and disk space.)

Full compose for a hotfix
=========================

Instead of using this ``hotfix-ubuntu`` script, you may want to generate a full compose.

#. Enter the rhcs-metadata Git clone, and ensure your testing branch is
   up-to-date::

     cd ~/dev/rhcs-metadata
     git checkout testing
     git fetch && git reset --hard origin/testing

#. We are basing this hotfix on top of ceph-2.2, so let's copy that
   configuration::

     cp ceph-2-ubuntu.conf ceph-2.2-ubuntu-hotfix-bz1445891.conf
     git add ceph-2.2-ubuntu-hotfix-bz1445891.conf

#. Determine the builds list upon which to base this hotfix.  Look at
   all the builds lists and determine which one would be appropriate. In
   our case, we want to start from the build lists that most-recently
   shipped to customers.

#. Create your new hotfix build lists::

     cp builds-ceph-2.2-27750-trusty.txt builds-ceph-2.2-hotfix-bz1445891-trusty.txt
     cp builds-ceph-2.2-27750-xenial.txt builds-ceph-2.2-hotfix-bz1445891-xenial.txt
     git add builds-ceph-2.2-hotfix-bz1445891-{trusty,xenial}.txt

#. Set ``product_version`` in
   ``ceph-2.2-ubuntu-hotfix-bz1445891.conf`` from ``2`` to ``2.2``

#. Set the new ``builds`` lists text files in
   ``ceph-2.2-ubuntu-hotfix-bz1445891.conf``.
   
#. Edit the new hotfix build lists to have the hotfix NVR 
   (e.g.: ceph_10.2.5-28.1.bz1445891redhat1xenial) and remove the 
   existing ceph NVR::

     vi builds-ceph-2.2-hotfix-bz1445891-trusty.txt
     vi builds-ceph-2.2-hotfix-bz1445891-xenial.txt

#. When the Jenkins builds are done and present in chacra, commit
   everything and push to rhcs-metadata.git's origin::

     git commit -a
     git push origin testing

#. Jenkins will not automatically merge "testing" to "master", so do
   that by hand::

     git checkout master
     git merge tesing --ff-only
     git push origin master

#. Open a ticket with rel-eng to generate and beta-sign this compose. Be
   sure to mention the exact .conf filename
   (``ceph-2.2-ubuntu-hotfix-bz1445891.conf``) in the ticket.
   https://projects.engineering.redhat.com/projects/RCM/issues
