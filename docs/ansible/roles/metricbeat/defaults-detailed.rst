.. Copyright (C) 2022 Maciej Delmanowski <drybjed@gmail.com>
.. Copyright (C) 2022 DebOps <https://debops.org/>
.. SPDX-License-Identifier: GPL-3.0-only

Default variable details
========================

Some of the ``debops.metricbeat`` default variables have more extensive
configuration than simple strings or lists, here you can find documentation and
examples for them.


.. _metricbeat__ref_configuration:

metricbeat__configuration
-------------------------

The ``metricbeat__*_configuration`` variables define the contents of the
:file:`/etc/metricbeat/metricbeat.yml` configuration file. Each variable contains
a list of YAML dictionaries; each dictionary defines a part of the
configuration which gets merged together during Ansible execution.

You can read the `Metricbeat configuration documentation`__ to learn more about
configuring Metricbeat itself.

.. __: https://www.elastic.co/guide/en/beats/metricbeat/current/configuring-howto-metricbeat.html

Examples
~~~~~~~~

Configure Metricbeat to output its data to Elasticsearch on another host:

.. code-block:: yaml

   metricbeat__configuration:

     - name: 'output_elasticsearch'
       config:
         output.elasticsearch:
           hosts:
             - 'elasticsearch.example.org:9200'

Configure Elasticsearch output, but over an encrypted connection (requires
X-Pack support) using certificates managed by the :ref:`debops.pki` role. The
access to the cluster is protected by a password, stored in the Metricbeat
keystore:

.. code-block:: yaml

   metricbeat__configuration:

     - name: 'output_elasticsearch'
       config:
         output.elasticsearch:
           hosts:
             - 'https://elasticsearch.example.org:9200'
           ssl:
             certificate_authorities: '/etc/pki/realms/domain/CA.crt'
             certificate: '/etc/pki/realms/domain/default.crt'
             key: '/etc/pki/realms/domain/default.key'
           password: '${ELASTIC_PASSWORD}'

The :envvar:`metricbeat__original_configuration` variable contains the
configuration that comes with the ``metricbeat`` APT package re-implemented for
consumption by the role. The :envvar:`metricbeat__default_configuration`
variable contains some additional configuration enabled by default.

Syntax
~~~~~~

Each configuration entry is a YAML dictionary with specific parameters:

``name``
  Required. An identifier for a particular configuration entry, not used
  otherwise. The configuration entries with the same ``name`` parameter
  override each other.

``config``
  Required. A dictionary which holds the Metricbeat configuration written in
  YAML. The ``config`` values from different configuration entries are merged
  recursively using the ``combine`` Ansible filter into a final YAML document.

  YAML keys can be specified in a tree-like structure:

  .. code-block:: yaml

     output:
       elasticsearch:
         hosts:
           - 'elasticsearch.example.org:9200'

  Or, they can be defined on a single line, separated by dots:

  .. code-block:: yaml

     output.elasticsearch.hosts: [ 'elasticsearch.example.org:9200' ]

  The ``combine`` Ansible filter does not automatically expand the dot-notation
  to a tree-like structure. Therefore it's important to use the same style
  thruought the configuration, otherwise the final YAML document will have
  duplicate entries.

``state``
  Optional. If not specified or ``present``, the configuration will be included
  in the generated :file:`/etc/metricbeat/metricbeat.yml` configuration file.
  if ``absent``, the configuration will not be included in the final file. If
  ``ignore``, the entry will not be evaluated by Ansible during execution.


.. _metricbeat__ref_snippets:

metricbeat__snippets
--------------------

The ``metricbeat__*_snippets`` variables define the placement and contents of
various :file:`*.yml` files under the :file:`/etc/metricbeat/` directory. The
files can include Metricbeat configuration in YAML format.

Examples
~~~~~~~~

Configure Metricbeat to gather the :command:`nginx` metrics and send them to
Elasticsearch:

.. code-block:: yaml

   metricbeat__snippets:

     - name: 'modules.d/nginx.yml'
       config:
         - module: 'nginx'
           metricsets:
             - 'stubstatus'
           period: '10s'
           hosts: [ 'http://127.0.0.1' ]
           server_status_path: 'nginx_status'

You can find more example configurations in the
:envvar:`metricbeat__default_snippets` variable.

Syntax
~~~~~~

Each configuration entry is a YAML dictionary with specific parameters:

``name``
  Required. Path of the configuration file, relative to the
  :file:`/etc/metricbeat/` directory, with all needed subdirectories. The
  ``name`` parameter is also used as an identifier, entries with the same
  ``name`` parameter override each other in order of appearance.

  Metricbeat includes by default all files in the :file:`modules.d/*.yml` path.
  Don't use the :file:`metricbeat.yml` as the filename, otherwise you will
  override the main configuration file.

``config``
  Required. A dictionary which holds the Metricbeat configuration written in
  YAML. The value can either be a dictionary or a list of dictionaries, the
  result in the generated file will always be a list.

``state``
  Optional. If not specified or ``present``, the configuration file will be
  generated. If ``absent``, the configuration file will not be generated, and
  an existing file will be removed. If ``ignore``, the entry will not be
  evaluated by Ansible during execution.

``comment``
  Optional. Comment to be included at the top of the generated file.

``mode``
  Optional. Specify the filesystem permissions of the generated file. If not
  specified, ``0600`` will be used by default.


.. _metricbeat__ref_keys:

metricbeat__keys
----------------

The ``metricbeat__*_keys`` variables define the contents of the `Metricbeat
keystore`__ used to keep confidental data like passwords or access tokens. The
keys can be referenced in the Metricbeat configuration files using the
``${secret_key}`` syntax.

.. __: https://www.elastic.co/guide/en/beats/metricbeat/current/keystore.html

Examples
~~~~~~~~

Add an Elasticsearch password used for access over a secure connection. The
password is retrieved from the :file:`secret/` directory on the Ansible
Controller, managed by the :ref:`debops.secret` Ansible role:

.. code-block:: yaml

   metricbeat__keys:

     - ELASTIC_PASSWORD: '{{ lookup("file", secret + "/elastic-stack/elastic/password") }}'

Update an existing key with new content (presence of the ``force`` parameter
will update the key on each Ansible run):

.. code-block:: yaml

   metricbeat__keys:

     - name: 'ELASTIC_PASSWORD'
       value: 'new-elasticsearch-password'
       force: True

Remove a key from the Metricbeat keystore:

.. code-block:: yaml

   metricbeat__keys:

     - name: 'ELASTIC_PASSWORD'
       state: 'absent'

Syntax
~~~~~~

Each key entry is defined by a YAML dictionary. The keys can be defined using
a simple format, with dictionary key being the secret key name, and its value
being the secret value. In this case you should avoid the ``name`` or ``value``
as the secret keys.

Alternatively, secret keys can be defined using YAML dictionaries with specific
parameters:

``name``
  Required. Name of the secret key to store in the Metricbeat keystore.

``value``
  Optional. A string with the value which should be stored under a given key.

``state``
  Optional. If not specified or ``present``, the key will be inserted into the
  keystore. If ``absent``, the key will be removed from the keystore.

``force``
  Optional, boolean. If present and ``True``, the specified key will be updated
  in the keystore.
