***********************************
Bundle Installation on Debian
***********************************

perfSONAR combines various sets of measurement tools and services. For perfSONAR 4.4 we provide the whole perfSONAR toolkit as Debian packages for six different architectures.  This should enable you to deploy a full perfSONAR node on one of the following distributions:

* Debian 9 Stretch
* Ubuntu 18 Bionic Beaver

Partial support of perfSONAR, i.e. only ``perfsonar-testpoint`` bundle at the moment, is also available on the following distributions:

* Debian 10 Buster

Debian meta packages are available to install the bundles described in :doc:`install_options`. The remaining sections of this document detail the steps required for installing these bundles.


System Requirements
===================

* **Architecture:** We provide Debian packages for 6 different architectures:

  * 32-bit (i386)
  * 64-bit (amd64)
  * ARMv4t and up (armel) (only for Debian)
  * ARMv7 and up (armhf)
  * ARM64 (arm64)
  * PPC64 (ppc64el)

* **Operating System:**  Any system running a Debian 9 or Ubuntu 18 server OS is supported.  Other Debian flavours derived from Debian 9 or Ubuntu 18 might work too but are not officially supported.

* See :doc:`install_hardware` for hardware requirements and more.

.. note:: Installing a graphical/desktop environment with perfSONAR is not supported.  These environments generally come with a Network Manager that conflicts with the way that perfSONAR is tuning the network interface parameters.  We recommend doing only server grade OS installs.

.. _install_debian_installation:

Installation 
============

.. _install_debian_step1:

Step 1: Configure APT
---------------------
All you need to do is to configure the perfSONAR Debian repository source, along with our signing key, on your Debian/Ubuntu machine. **You will need to follow the steps below as privileged user**::

    cd /etc/apt/sources.list.d/
    curl -o perfsonar-release.list http://downloads.perfsonar.net/debian/perfsonar-release.list
    curl http://downloads.perfsonar.net/debian/perfsonar-official.gpg.key | apt-key add -
   
* **Ubuntu only**. Additionnaly, if you're running a stripped down Ubuntu installation, you might need to enable the universe repository.  This is done with the following command::

    add-apt-repository universe

Then refresh the packages list so APT knows about the perfSONAR packages::

    apt update


.. _install_debian_step2:

Step 2: Install a Bundle 
------------------------ 
**Choose one** of the following bundles and see :doc:`install_options` page for more information about what these bundles are.

* **perfSONAR Tools**::

    apt install perfsonar-tools

* **perfSONAR Test Point**::

    apt install perfsonar-testpoint  

  During the installation process, you'll be asked to choose a password for the pscheduler database.

* **perfSONAR Core**::

    apt install perfsonar-core

  During the installation process, you'll be asked to choose a password for the pscheduler and the esmond databases.

* **perfSONAR Central Management**::

    apt install perfsonar-centralmanagement

  During the installation process, you'll be asked to choose a password for the esmond database.

* **perfSONAR Toolkit**::

    apt install perfsonar-toolkit

  During the installation process, you'll be asked to choose a password for the pscheduler and the esmond databases.

.. note:: On Debian 10, only the ``perfsonar-tools`` and the ``perfsonar-testpoint`` bundles are supported at the moment.

Optional Packages
++++++++++++++++++
In addition to any of the bundles above you may also **optionally** choose to install one or more of our add-on packages (these are automatically added on the perfsonar-toolkit bundle):

     * ``apt install perfsonar-toolkit-ntp`` - Automatically detects closest NTP servers and sets them in ntp.conf
     * ``apt install perfsonar-toolkit-security`` - Adds default firewall rules and installs fail2ban
     * ``apt install perfsonar-toolkit-servicewatcher`` - Adds a cron job that checks if services are still running
     * ``apt install perfsonar-toolkit-sysctl`` - Adds default sysctl tuning settings
     * ``apt install perfsonar-toolkit-systemenv-testpoint`` - Configures auto-update and set some default logging locations

You may also run the command below to get everything listed above on **perfsonar-testpoint** and **perfsonar-core** bundles::

    /usr/lib/perfsonar/scripts/install-optional-packages.py

.. note:: On a **perfsonar-centralmanagement** system you probably only want the optional **perfsonar-toolkit-servicewatcher** package to be installed.

Reducing installation size
++++++++++++++++++++++++++
If you want to reduce the perfSONAR installation size as much as possible, you can call ``apt`` with the ``--no-install-recommends`` option.  This will prevent Debian recommended packages to be automatically installed (you can also configure this globaly in the APT configuration files with the statement ``APT::Install-Recommends "0";``).  This can become useful when you want to install the perfsonar-testpoint bundle with the less overhead possible.


.. _install_debian_step3:

Step 3: Verify NTP and Tuning Parameters 
----------------------------------------- 
*Step 3 can be ignored for perfsonar-toolkit package installation as its instructions are included and run automatically*

* **NTP Tuning**

  - **Auto-select NTP servers based on proximity**
    
    The Network Time Protocol (NTP) is required by the tools in order to obtain accurate measurements. Some of the tools such as OWAMP will give correct results unless NTP is running. If the optional package `perfsonar-toolkit-ntp` was installed this has already been done for you, but if you want to re-run manually::

        /usr/lib/perfsonar/scripts/configure_ntpd new
        service ntp restart

  You can also configure your own set of NTP servers if you want.

  You can verify if NTP is running with the following command::

        /usr/sbin/ntpq -p  

* **System Tuning**
  
  It is important to make sure that your host is properly tuned for maximum TCP performance on the WAN. You should verify that htcp, not reno, is the default TCP congestion control algorithm, and that the maximum TCP buffers are big enough for your paths of interest.  

  If you have installed the `perfsonar-toolkit-sysctl` package, all should be ready for you, but if you want to rerun manually::

    /usr/lib/perfsonar/scripts/configure_sysctl

  Please refer to `linux host tuning <http://fasterdata.es.net/host-tuning/linux/>`_ for more information.


.. _install_debian_step4:

Step 4: Firewall and Security Considerations 
--------------------------------------------- 
If you have installed the `perfsonar-toolkit-security` package, then your iptables are already configured with our default rules.  The package also installs fail2ban.

If you would like to configure the rules manually, then please review the `document here <http://www.perfsonar.net/deploy/security-considerations/>`_ on the ports that need to be open.

    The `perfsonar-toolkit-security` package uses `firewalld` to manage the firewall rules.

Additionally, pscheduler allows you to limit the parameters of tests such as duration and bandwidth based on the requesters IP address. It does this through the file ``pscheduler/limits.conf``. 
ESnet provides a file containing all R&E subnets, which is updated nightly. Instructions on how to download this file and configure pScheduler to use it are described on the page :doc:`manage_limits`.

Note that the `perfsonar-toolkit-security` package is automatically included in the `perfsonar-toolkit` bundle.

.. _install_debian_step5:

Step 5: Auto updates
--------------------
If you have installed the `perfsonar-toolkit-systemenv-testpoint` package, then you're all set for the auto-updates of perfSONAR packages and security fix for your OS.

To ensure you always have the most current and hopefully most secure packages you can install ``unattended-upgrades``. You’ll need to configure it to actually install the available updates with the following commands:
::

    apt install unattended-upgrades
    echo 'APT::Periodic::Update-Package-Lists "1";' > /etc/apt/apt.conf.d/60unattended-upgrades-perfsonar
    echo 'APT::Periodic::Unattended-Upgrade "1";' >> /etc/apt/apt.conf.d/60unattended-upgrades-perfsonar
    echo 'APT::Periodic::AutocleanInterval "31";' >> /etc/apt/apt.conf.d/60unattended-upgrades-perfsonar
    echo 'Unattended-Upgrade::Origins-Pattern:: "origin=perfSONAR";' >> /etc/apt/apt.conf.d/60unattended-upgrades-perfsonar

A cronjob will automatically install security updates from Debian/Ubuntu and new packages present in the perfsonar release repository every night. A trace of all updates applied will be stored in ``/var/log/unattended-upgrades/unattended-upgrades.log``.

Full perfSONAR toolkit upgrades (i.e. upgrade to new major versions) might still need a manual intervention to properly conclude, but we will then announce that through our usual communication channels.

.. note:: Automatic updates are enabled by default in the perfSONAR Toolkit.

.. _install_debian_step6:

Step 6: Service Watcher
------------------------
The `perfsonar-toolkit-servicewatcher` installs scripts that check if pscheduler, owamp, databases and other processes are running and restarts if they have stopped unexpectedly. 

The install automatically configures cron to run the service_watcher regularly.

To run the script manually, run::

  /usr/lib/perfsonar/scripts/service_watcher

.. _install_debian_step7:

Step 7: Register your services 
------------------------------- 
Note: this step can be done through the web interface if the perfsonar-toolkit bundle was installed. See :doc:`manage_admin_info`.

No actual configuration is required but filling fields such as administrator_email, site_name, city, country, latitude, longitude, etc. are **highly recommended**. You can add these by removing the leading `#` of any property and filling it out with a proper value for your host. Changes will be picked-up automatically without need for any restarts.

.. _install_debian_step8:

Step 8: Starting your services 
------------------------------- 
You can start all the services by rebooting the host since all are configured to run by default. In order to check services status issue the following commands::
    
    service pscheduler-scheduler status
    service pscheduler-runner status
    service pscheduler-archiver status
    service pscheduler-ticker status
    service owamp-server status
    service perfsonar-lsregistrationdaemon status

If they are not running you may start them with appropriate service commands as a root user. For example::

    service pscheduler-scheduler start
    service pscheduler-runner start
    service pscheduler-archiver start
    service pscheduler-ticker start
    service owamp-server start
    service perfsonar-lsregistrationdaemon start

Note that you may have to wait a few hours for NTP to synchronize your clock before (re)starting owamp-server.

Configuration
=============

Configuring Central Management
-------------------------------
If you installed the perfsonar-centralmanagement bundle see the following documents:

* :doc:`multi_ma_install`
* :doc:`psconfig_publish`
* :doc:`psconfig_maddash_agent`
* :doc:`maddash_intro`


Configuring perfSONAR through the web interface
------------------------------------------------
After installing the perfsonar-toolkit bundle, you can refer to the general perfSONAR configuration from :doc:`install_config_first_time`.

Upgrading from 4.3.x (or 4.2.x)
========================================
If you had installed a perfSONAR 4.3.x (or 4.2.x) bundle and you now want to upgrade to perfSONAR 4.4, you'll have to follow the instructions here below.  This will work for all Debian and Ubuntu versions supported on both releases, i.e. Debian 9, Debian 10 and Ubuntu 18.

Upgrade the perfSONAR installation
----------------------------------
If you have auto-update enabled and already using the ``perfsonar-release.list`` APT source file (as was instructed when installing 4.3 or 4.2), you should receive the 4.4 upgrade automatically. However, because of some dependency changes and repository name change, the full upgrade need to be done manually.

If you don't use the auto-update feature, to upgrade your perfsonar installation, you need to run::

   apt update
   apt upgrade

The measurements and the measurement archives that you already have defined in your 4.3.x (or 4.2.x) installation will be migrated to the 4.4 toolkit automatically.

If you are upgrading from 4.2, you'll need to run the command ``apt dist-upgrade`` because of a change in the depencies created by the move to Python 3.

.. note:: You might see ``apt`` issuing a warning about conflicting distribution with a message like ``W: Conflicting distribution: http://downloads.perfsonar.net/debian perfsonar-release InRelease (expected perfsonar-4.3 but got perfsonar-4.4)``  This is expected and can be ignored because you indeed are upgrading from 4.3 to 4.4.

Upgrade to another bundle
-------------------------
If you want to move from the `perfsonar-testpoint` bundle to another bundle that we provide for Debian, you can do so by following the instructions above from :ref:`install_debian_step2`.

Upgrade from Ubuntu 16 to Ubuntu 18
-----------------------------------
If you have a perfSONAR host running Ubuntu 16 and you want to upgrade it to 18, we recommend you to follow the following steps:

* Upgrade Ubuntu 16 to Ubuntu 18 (following official instructions, here are `Bionic Upgrades notes <https://help.ubuntu.com/community/BionicUpgrades>`_)
* Reboot your system unless already done in previous step.
* Run apt-get update; apt-get dist-upgrade to get the latest version of perfSONAR.
* Reboot your system one last time.

Alternatively, do a fresh installation of perfSONAR on Ubuntu 18.

