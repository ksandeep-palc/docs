.. role:: raw-html-m2r(raw)
   :format: html


psConfig Web Administrator (PWA)
================================

psConfig Web-based administration GUI and tools to publish generated meshconfig/psconfig output


.. image:: images/pwa/pwa_install.png
   :target: images/pwa/pwa_install.png
   :alt: Alt text


Installation
------------

VM Host
^^^^^^^

To install PWA, you will need a VM with any OS that supports Docker; such as CentOS7

Minimum resource requirements are..


* 4-6 CPUs
* 4G memory
* 16G disk

Docker Engine
^^^^^^^^^^^^^

Read the official `docker installation doc <https://docs.docker.com/engine/installation/>`_ for more information. For CentOS 7, the Docker version from the CentOS Extras repo will work. For CentOS 6, the CentOS version might work, or you might need to try the version from the Docker repo.

For CentOS7 as root:

.. code-block:: bash

   yum install -y docker

Before you start the docker engine, you might want to add any VM specific configuration. For example, your VM might be using /usr/local as a primary partition for your VM. If so, you should have something like following..

.. code-block:: bash

   mkdir /etc/docker

``/etc/docker/daemon.json``

.. code-block:: json

   {
           "graph": "/usr/local/docker"
   }

Enable & start the docker engine.

.. code-block:: bash

   $ systemctl enable docker
   $ systemctl start docker

You should install logrotate for docker container log

/etc/logrotate.d/docker-container

.. code-block:: bash

   /var/lib/docker/containers/*/*.log {
     rotate 7
     daily
     compress
     size=1M
     missingok
     delaycompress
     copytruncate
   }

Configuration
^^^^^^^^^^^^^

**Interaction with other web applications:** If you want to run PWA on a node that is already running other web applications, such as MadDash or the perfSONAR Toolkit web interface, you will need to do a couple things differently. See `Running alongside other web applications <pwa_running_alongside>`_

**Upgrading:** If you are upgrading from a legacy MCA instance, read `UPGRADING FROM MCA <pwa_upgrading_from_mca>`_

Before we start installing PWA, you should prepare your configuration files first. You can bootstrap it by
downloading and deploying PWA's default configuration files from git repo.

.. code-block:: bash

   wget https://github.com/perfsonar/psconfig-web/raw/master/deploy/docker/pwa.sample.tar.gz
   tar -xzf pwa.sample.tar.gz -C /etc


#. 
   For PWA

    ``/etc/pwa/index.js`` 


   * Edit defaults ``testspecs`` if necessary (\ ``meshconfig.defaults.testspecs``\ )
   * Update pub.url with the hostname that your PWA instance will be exposed as. The easiest way to do this is to replace :raw-html-m2r:`<pwa_hostname>` with the FQDN of your Docker host (removing the brackets).
   * Edit datasource section which determines which host you'd like to load from sLS to construct your host config, if applicable (if you are not running a private LS, this most likely does not apply to you)

#. 
   For Authentication Service

    ``/etc/pwa/auth/index.js``

    Update the hostname in the config by performing a search and replace in this file. Replace :raw-html-m2r:`<pwa_hostname>` with the hostname (FQDN) of the host that holds your docker containers (remove the brackets).

    Update ``from`` address to administrator's email address used to send email to confirmation new user accounts. You can do this by doing a search and replace in the file, replacing :raw-html-m2r:`<email_address>` with the full e-mail address you want to use (remove the brackets).

    If you'd like to skip email confirmation when user signup, simply comment out the whole email_confirmation section.

   .. code-block:: javascript

       exports.email_confirmation = {
           subject: 'psConfig Web Admin Account Confirmation',
           from: '<email_address>',  //most mail server will reject if this is not replyable address
       };

#. 
   For Nginx

    Nginx will expose various functionalities provides by various containers to the actual users. The default configuration should work, but if you need to modify the configuration, edit..

    ``/etc/pwa/nginx``

Host Certificates
~~~~~~~~~~~~~~~~~

You will need SSL certificates for https access. If these don't already exist, they will be created when you start up the ``sca-auth`` docker container.

In ``/etc/pwa/auth``\ , you should see your host certificate with following file names, or place them there if not. If you want to replace the automatically-generated certs with your own, copy them over these files.

.. code-block:: bash

   $ ls /etc/pwa/auth
   cert.pem 
   key.pem

If you are enabling x509 authentication, then you will also need ``trusted.pem``. This file contains list of all CAs that you trust and grant access to PWA.

..

   Unlike Apache, Nginx uses a single CA file for better performance.. so you have to join all .pem into a single .pem file.


Container Installation
^^^^^^^^^^^^^^^^^^^^^^

Now we have all configuration files necessary to start installing PWA services.


#. 
   First, create a docker network to group all PWA containers (so that you don't have --link them)

   .. code-block:: bash

       docker network create pwa

#. 
   Create mongoDB container. Use -v to persist data on host directory (/usr/local/data/mongo)

   .. code-block:: bash

       mkdir -p /usr/local/data
       docker run \
               --restart=always \
               --net pwa \
               --name mongo \
               -v /usr/local/data/mongo:/data/db \
               -d mongo

#. 
   Create SCA authentication service container. This service handles user authentication / account/user group management.

   .. code-block:: bash

       docker run \
           --restart=always \
           --net pwa \
           --name sca-auth \
           -v /etc/pwa/auth:/app/api/config \
           -v /usr/local/data/auth:/db \
           -d perfsonar/sca-auth

   ..

      sca-auth container will generate a few files under /config directory when it's first started, so don't mount it with ``ro``.
      I am persisting the user account DB on /usr/local/data/auth.


#. 
   Create PWA's main UI/API container.

   .. code-block:: bash

       docker run \
           --restart=always \
           --net pwa \
           --name pwa-admin1 \
           -v /etc/pwa:/app/api/config:ro \
           -d perfsonar/pwa-admin

#. 
   Create meshconfig publishers.

   .. code-block:: bash

       docker run \
           --restart=always \
           --net pwa \
           --name pwa-pub1 \
           -v /etc/pwa:/app/api/config:ro \
           -d perfsonar/pwa-pub

You can create as many pwa-pub containers as desired (make sure to use unique names ``pwa-pub1``\ , ``pwa-pub2``\ , etc..) based on available resource (mainly CPU) . 1 or 2 should be fine for most cases.

If you use more than 1 instance, please edit ``/etc/pwa/nginx/conf.d/pwa.conf`` to include all instances, like..

.. code-block:: javascript

   upstream pwapub {
       server pwa-pub1:8080;
       server pwa-pub2:8080;
       server pwa-pub3:8080;
   }


#. 
   Finally, we install nginx to expose these container via 80/443/9443

   .. code-block:: bash

       docker run \
           --restart=always \
           --net pwa \
           --name nginx \
           -v /etc/pwa/shared:/shared:ro \
           -v /etc/pwa/nginx:/etc/nginx:ro \
           -v /etc/pwa/auth:/certs:ro \
           -p 80:80 \
           -p 443:443 \
           -p 9443:9443 \
           -d nginx

Now you should see all 5 containers running.

.. code-block:: bash

   docker container list
   CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                                              NAMES
   42efd21ff7f1        perfsonar/pwa-pub     "node /app/api/mcp..."   18 seconds ago      Up 17 seconds       8080/tcp                                                           pwa-pub1
   ab3936c7ab8c        perfsonar/pwa-admin   "/start.sh"              19 seconds ago      Up 18 seconds       80/tcp, 8080/tcp                                                   pwa-admin1
   90cfbb8ba096        perfsonar/sca-auth    "/app/docker/start.sh"   24 seconds ago      Up 24 seconds       80/tcp, 8080/tcp                                                   sca-auth
   aa6471073c01        nginx               "nginx -g 'daemon ..."   11 hours ago        Up 11 hours         0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:9443->9443/tcp   nginx
   10fdf3b63e4f        mongo               "/entrypoint.sh mo..."   12 hours ago        Up 12 hours         27017/tcp                                                          mongo

Note: sometimes, docker containers will initially not have connectivity to the outside world. Usually this can be resolved by running ``systemctl restart docker``

Updating
^^^^^^^^

To update PWA containers to the latest version, stop/remove the current container. This example updates the pwa-admin image, but you might also need to do the same thing for ``pwa-pub`` and/or ``sca-auth``\ , as well.

.. code-block:: bash

   docker stop pwa-admin1
   docker rm pwa-admin1

Pull down the latest version using:

.. code-block:: bash

   docker pull perfsonar/pwa-admin1

Re-run the container using the same ``docker run ...`` command you used to start it.

Firewall
^^^^^^^^

Docker will take care of its own firewall rules, so you don't have to worry about opening ports manually. 

By default, following are the ports used by nginx container:


* 443 (For PWA administrative GUI)
* 80 (For PWA configuration publisher)
* 9443 (For x509 authentication to PWA administrative GUI)

Other Topics
============


* `Monitoring / Testing <pwa_monitoring>`_
* PWA provides a developer API -- see the `API DOC <pwa_api>`_

Reference
=========

Meshconfig parameters
http://docs.perfsonar.net/config_mesh.html
