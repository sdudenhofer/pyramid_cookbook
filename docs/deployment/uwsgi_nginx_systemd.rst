uWSGI + nginx + systemd
+++++++++++++++++++++++

This chapter provides an example for configuring `uWSGI <https://uwsgi-docs.readthedocs.io/en/latest/>`_, `nginx <https://nginx.org/en/docs/>`_, and `systemd <https://www.freedesktop.org/wiki/Software/systemd/>`_ for a Pyramid application.

Below you can find an almost production ready configuration. "Almost" because some ``uwsgi`` parameters might need tweaking to fit your needs.

An example systemd configuration file is shown here:

.. code-block:: ini
    :linenos:

    # /etc/systemd/system/pyramid.service

    [Unit]
    Description=pyramid app

    # Requirements
    Requires=network.target

    # Dependency ordering
    After=network.target

    [Service]
    TimeoutStartSec=0
    RestartSec=10
    Restart=always

    # path to app
    WorkingDirectory=/opt/env/wiki
    # the user that you want to run app by
    User=app

    KillSignal=SIGQUIT
    Type=notify
    NotifyAccess=all

    # Main process
    ExecStart=/opt/env/bin/uwsgi --ini-paste-logged /opt/env/wiki/development.ini

    [Install]
    WantedBy=multi-user.target

.. note:: In order to use the ``--ini-paste-logged`` parameter (and have logs from an application), `PasteScript <https://pypi.org/project/PasteScript/>`_ is required. To install, run:

    .. code-block:: bash

        pip install PasteScript

uWSGI can be configured in ``.ini`` files, for example:

.. code-block:: ini
    :linenos:

    # development.ini
    # ...

    [uwsgi]
    socket = /tmp/pyramid.sock
    chmod-socket = 666
    protocol = http

Save the files and run the below commands to start the process:

.. code-block:: bash
    
    systemctl enable pyramid.service
    systemctl start pyramid.service

Verify that the file ``/tmp/pyramid.sock`` was created.

Here are a few useful commands:

.. code-block:: bash

    systemctl restart pyramid.service # restarts app
    journalctl -fu pyramid.service # tail logs

Next we need to configure a virtual host in nginx. Below is an example configuration:

.. code-block:: nginx
    :linenos:

    # myapp.conf

    upstream pyramid {
        server unix:///tmp/pyramid.sock;
    }

    server {
        listen 80;
    
        # optional ssl configuration
        
        listen 443 ssl;
        ssl_certificate /path/to/ssl/pem_file;
        ssl_certificate_key /path/to/ssl/certificate_key;
        
        # end of optional ssl configuration
    
        server_name  example.com;

        access_log  /opt/env/access.log;

        location / {
            proxy_set_header        Host $http_host;
            proxy_set_header        X-Real-IP $remote_addr;
            proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header        X-Forwarded-Proto $scheme;

            client_max_body_size    10m;
            client_body_buffer_size 128k;
            proxy_connect_timeout   60s;
            proxy_send_timeout      90s;
            proxy_read_timeout      90s;
            proxy_buffering         off;
            proxy_temp_file_write_size 64k;
            proxy_pass http://pyramid;
            proxy_redirect          off;
        }
    }

A better explanation for some of the above nginx directives can be found in the cookbook recipe :doc:`nginx + pserve + supervisord <nginx>`.
