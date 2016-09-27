``hotfix-ubuntu``
=================

A tool to package up hotfixes for Red Hat Ceph Storage for Ubuntu.

This tool uses `rhcephpkg <https://github.com/red-hat-storage/rhcephpkg>`_ to
download hotfix builds from Red Hat's build storage system (`rhcephpkg
<https://github.com/red-hat-storage/rhcephpkg>`_), generate an apt repository
with those builds (with `reprepro <http://mirrorer.alioth.debian.org/>`_), and
assemble everything into a single tarball for customers.

Example
=======

(prerequisite: Install and configure rhcephpkg and reprepro)

Clone from Git and run it::

  git clone https://github.com/ktdreyer/hotfix-ubuntu
  cd hotfix-ubuntu
  ./hotfix-ubuntu <bug-id> <build-nvr>

This will create a "hotfix-bzXXXX.tar.gz" in your local working directory.
