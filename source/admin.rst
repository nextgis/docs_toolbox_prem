Administrator manual for NextGIS Toolbox on-premise
=========================================================

.. _tbop_connection:

Select connection addresses
----------------------------

Toolbox On-Premise uses two endpoints: HTTP (or HTTPS):

* Users interact with the software via Web interface and API. The default value is ``http://server.example.com:58347`` where server.example.com is the DNS name of the server where the software is deployed. If strictly necessary, the server IP address can be used instead of server.example.com.

* The Buildbot interface is an auxiliary address, not used during normal operation, but might be required to allow the support team to resolve various issues. The default value is ``http://hostname.example.com:61978``.

If your IT infrastructure allows for it, it is recommended to set up a reverse proxy for TLS encryption and using HTTPS. It is especially important if the software is to be accessed not just from the local network, but also from the Internet. In that case the addresses of the entry points depend on the settings of the reverse proxy. The recommended parameters are:

* Toolbox Web UI and API - https://toolbox.example.com
* Buildbot interface - https://avral.example.com

Contact your IT department to choose addresses you wish to use and note them down, you'll need them later. The reverse proxy is set up by the client's IT department, it is not a responsibility of NextGIS company. The required parameters are cited below using Nginx as example.

::

   TOOLBOX_URL
   BUILDBOT_URL

.. _tb_ngid:

NextGIS ID authorization
-------------------------

Toolbox On-Premise uses NextGIS ID On-Premise as an authorization server. It is supplied as part of NextGIS Web On-Premise. First find our the address of NextGIS ID On-Premise connection. Make sure you can log in as administrator. The address may be something like https://ngid.example.com or http://server.example.com:8081. Note the connection address, you'll need it later::

   NEXTGISID_URL

Add the TOOLBOX_URL variable to the the x-shared section of the docker-compose.yaml file to get the following::

   x-shared: &shared
     NEXTGISID_URL: "https://ngid.example.com"
     NEXTGISWEB_URL: "https://ngw.example.com"
     TOOLBOX_URL: "https://toolbox.example.com"
     NEXTGISID_INSTANCE_NGID: "a73ed33d-7e04-4405-a1fc-90188093268e"
     NEXTGISID_ADMINISTRATOR_NGID: "cc533d32-6123-4d5f-8ffe-80e0135a54b6"

On the server where NextGIS ID On-Premise (NextGIS Web On-Premise) is installed go to `` /srv/ngwdocker`` and edit the file docker-compose.yaml (for example in a text editor). Then restart the stack. Note down the value of the NEXTGISID_ADMINISTRATOR_NGID variable from the x-shared section.

::

   $ cd /srv/ngwdocker
   $ nano docker-compose.yaml
   $ docker compose up -d

::

   NEXTGISID_ADMINISTRATOR_NGID

.. _tbop_docker_setup:

Install and configure Docker
------------------------------

If the server does not yet have Docker Engine and Docker Compose installed, first you need to install them or update them to the latest versions:

* `Docker Engine <https://docs.docker.com/engine/install/>`_
* `Docker Compose <https://docs.docker.com/compose/install/linux/>`_

To get the images log in to NextGIS Container Registry with the username (example) and password (sesame) provided by NextGIS::

   $ docker login cr.nextgis.com -u example -p sesame
   Login Succeeded

If the software is deployed to a server without Internet access, contact support for a single-file image archive instead. You'll need to transfer it to the server and load the images using 'docker load' command.

Toolbox On-Premise needs Docker Swarm. Initialize it after installation by running the following::

   $ docker swarm init


.. _tbop_setup:

Install NextGIS Toolbox
---------------------------

On the server where you plan to deploy Toolbox On-Premise, create the ``/srv/toolbox`` directory, then go to it, download the configuration template (``docker-compose-25.12.0.tar.bz2``, where 25.12.0 is the current version) and unpack it. If the server does not have Internet access, download the file on another PC and transfer it to the server.

::

  $ mkdir /srv/toolbox
  $ cd /srv/toolbox
  $ wget https://nextgis.com/onpremise/toolbox/docker-compose-25.12.0.tar.bz2
  $ tar jxf docker-compose-25.12.0.tar.bz2

Generate a password for Buildbot and note it down, you'll need it later:

::

  $ tr -dc A-Za-z0-9 < /dev/urandom | head -c 12; echo
  ezn81wfERYOW

::

  TOOLBOX_BUILDBOT_SECRET


Edit the .env file in a text editor, filling in the values of the variables you'd noted down. In the BUILDBOT_REGISTRY_AUTH variable, enter the user name and password (separated by a colon) for the NextGIS Container Registry connection. In the end you should get something like this::

  IMAGE_VERSION=25.12.0
  IMAGE_BASE=cr.nextgis.com/toolbox
  COMPOSE_BIND=0.0.0.0
  TOOLBOX_URL=https://toolbox.example.com
  BUILDBOT_URL=https://avral.example.com
  NEXTGISID_URL=https://ngid.example.com
  NEXTGISID_ADMINISTRATOR_NGID=7f16c028-df44-457b-be6d-cd9075fad034
  TOOLBOX_BUILDBOT_SECRET=ezn81wfERYOW
  BUILDBOT_REGISTRY_AUTH=example:sesame

After that you can launch the Docker Compose stack. We recommend launching postgres service first, then after about 30 seconds launch the rest::

  $ docker compose up -d postgres && sleep 30
  [+] Running 1/1
   ✔ postgres Pulled                                    1.0s
  [+] Running 3/3
   ✔ Network toolbox_default         Created            0.1s
   ✔ Volume "toolbox_data_postgres"  Created            0.0s
   ✔ Container toolbox-postgres-1    Started            0.3s
  $ docker compose up -d
  [+] Running 3/3
   ✔ background Pulled                                  1.6s
   ✔ app Pulled 1.6s
   ✔ docker Pulled                                      1.6s
   ✔ buildbot Pulled                                    1.6s
  [+] Running 5/5
   ✔ Volume "toolbox_data_storage"   Created            0.0s
   ✔ Container toolbox-postgres-1    Running            0.0s
   ✔ Container toolbox-background-1  Started            0.3s
   ✔ Container toolbox-app-1         Started            0.3s
   ✔ Container toolbox-docker-1      Started            0.3s
   ✔ Container toolbox-buildbot-1    Started            0.3s

This completes the installation. If you use HTTPS, next configure the reverse proxy server. Otherwise proceed to operability check.


.. _tbop_proxy:

Recommendations for reverse proxy setup
---------------------------------------------------

To use HTTPS encryption we recommend setting up a reverse proxy server based on Nginx. For reference here's a fragment of the configuration file for toolbox.example.com:

.. code-block::

  server {
      server_name toolbox.example.com;
      # Server directives: listen, ssl_* etc
  
      location / {
          client_max_body_size 2G;
  
          proxy_http_version 1.1;
          proxy_pass http://127.0.0.1:58347;
          proxy_set_header Host $http_host;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection $proxy_connection;
          proxy_set_header X-Forwarded-Proto $scheme;
          proxy_set_header X-Forwarded-For $remote_addr;
      }
  }

The client_max_body_size directive defines the maximum size of the uploaded file (2 GiB in the example), it is needed only for the web interface (not needed for Buildbot).

.. _tbop_check:

Operability check
---------------------------

In a browser open the Toolbox Web interface using the URL you chose (TOOLBOX_URL). You should see the main page with the list of tools and "Sign in" button in the top right corner.

Press **Sign in**, you'll be redirected to NextGIS ID On-Premise. You may need to enter the password for administrator. After that you should be redirected back to the Toolbox interface. In the top right corner the current user's username should be displayed.

Select the **Hello, World** tool and try to run it. The first run of a tool may take longer because that's when the Docker tool image is downloaded from the NextGIS Container Registry.

Try to run **Convert vector layer** using the test data. It allows to test uploading the input and downloading the output file.
