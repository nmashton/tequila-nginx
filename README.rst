tequila-nginx
=============

This repository holds an `Ansible <http://www.ansible.com/home>`_ role
that is installable using ``ansible-galaxy``.  This role contains
tasks used to install and set up an nginx webserver for a Django
deployment.  It exists primarily to support the `Caktus Django project
template <https://github.com/caktus/django-project-template>`_.

More complete documenation can be found in `caktus/tequila
<https://github.com/caktus/tequila>`_.


License
-------

This Ansible role is released under the BSD License.  See the `LICENSE
<https://github.com/caktus/tequila-nginx/blob/master/LICENSE>`_ file
for more details.


Contributing
------------

If you think you've found a bug or are interested in contributing to
this project check out `tequila-nginx on Github
<https://github.com/caktus/tequila-nginx>`_.

Development sponsored by `Caktus Consulting Group, LLC
<http://www.caktusgroup.com/services>`_.


Installation
------------

Create an ``ansible.cfg`` file in your project directory to tell
Ansible where to install your roles (optionally, set the
``ANSIBLE_ROLES_PATH`` environment variable to do the same thing, or
allow the roles to be installed into ``/etc/ansible/roles``) ::

    [defaults]
    roles_path = deployment/roles/

Create a ``requirements.yml`` file in your project's deployment
directory ::

    ---
    # file: deployment/requirements.yml
    - src: https://github.com/caktus/tequila-nginx
      version: 0.1.0

Run ``ansible-galaxy`` with your requirements file ::

    $ ansible-galaxy install -r deployment/requirements.yml

or, alternatively, run it directly against the url ::

    $ ansible-galaxy install git+https://github.com/caktus/tequila-nginx

The project then should have access to the ``tequila-nginx`` role in
its playbooks.


Variables
---------

The following variables are made use of by the ``tequila-nginx``
role:

- ``project_name`` **required**
- ``domain`` **required**
- ``additional_domains`` **default:** empty list
  This is used to configure additional domains that should be allowed
  through to serve content from.
- ``redirect_domains`` **default:** empty list
  This is to configure subdomains to do permanent redirects back to
  the main ``domain`` host, e.g. redirecting www to non-www.
- ``root_dir`` **default:** ``"/var/www/{{ project_name }}"``
- ``public_dir`` **default:** ``"{{ root_dir }}/public"``
- ``static_dir`` **default:** ``"{{ root_dir }}/public/static"``
- ``media_dir`` **default:** ``"{{ root_dir }}/public/media"``
- ``nginx_conf_template`` **default:** ``"nginx.conf.j2"``
- ``client_max_body_size`` **default:** ``1m``
- ``ssl_dir`` **default:** ``"{{ root_dir }}/ssl"``
- ``force_ssl`` **default:** ``true``
- ``http_auth`` **default:** empty list
- ``auth_file`` **default:** ``"{{ root_dir }}/.htpasswd"``
- ``dhparams_file`` **default:** ``"{{ ssl_dir }}/dhparams.pem"``
- ``dhparams_numbits`` **default:** ``2048``
- ``cert_source`` **required, values:** ``'none'``, ``'letsencrypt'``, ``'selfsign'``, ``'provided'``
- ``admin_email`` **required if cert_source = letsencrypt**
- ``ssl_cert`` **required if cert_source = provided**
- ``ssl_key`` **required if cert_source = provided**
- ``certbot_certs`` **required if cert_source = letsencrypt** (see ``geerlingguy.certbot`` docs for format)
- ``use_memcached`` **default:** ``true``
- ``app_minions`` **required:** combined list of web servers and celery worker servers
- ``project_port`` **default:** 8000 - what port to proxy Django requests to
- ``nginx_http_port`` **default:** ``80``
  Override the port nginx listens on without SSL. Can be used if a load balancer
  is listening on port 80.
- ``nginx_https_port`` **default:** ``443``
  Override the port nginx listens on with SSL. Can be used if a load balancer
  is listening on port 443.

The ``app_minions`` variable can be constructed from Ansible's
inventory information, like ::

    app_minions: "{{ groups['web'] | union(groups['worker']) }}"

in one of your project's variable files.

The ``http_auth`` variable is meant to be a list of dicts, where each
dict contains a login and password.  So, a typical definition in an
actual project might look like ::

    http_auth:
      - login: user1
        password: password1
      - login: user2
        password: password2

The ``nginx_conf_template`` variable is to allow the template that
ships with this role to be overridden, if more complex directives need
to be supported.


Let's Encrypt / Certbot
-----------------------

If the ``'letsencrypt'`` option is chosen for ``cert_source``, tasks
in this role will download and install certbot-auto, and run it using
the ``certonly`` option.  This will put a token file up on your web
server under the .well-known/ directory, so you will need to have a
correctly configured and running nginx instance in order for the Let's
Encrypt validation server to be able to read this file.  One way of
doing this when bootstraping a clean server is to do your first deploy
using ``cert_source: none`` and ``force_ssl: false``, then switch to
``cert_source: letsencrypt`` (while still keeping ``force_ssl:
false``) for a second deployment.  If this works and a certificate is
successfully acquired, you can then safely switch to ``force_ssl:
true`` for subsequent deployments.

Finally, a cron job will be added to fire off the ``certbot-auto
renew`` action every day.

In previous versions of tequila-nginx, the certificate renewal was set
to occur monthly.  If your server has an older version of this
cron job under the name ``renew_certbot``, Ansible will replace it
with one with the new parameters when you deploy.

If you still have a yet older version of the cron job under the name
``renew_letsencrypt``, you can clear it out with an ad-hoc command
like this::

    $ ansible web -i deployment/environments/staging/inventory -m cron -a "name=renew_letsencrypt cron_file=letsencrypt state=absent"
