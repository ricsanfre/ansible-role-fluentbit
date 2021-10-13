Ansible Role: Fluentbit
=========

Fluentbit installation and configuration role.


Requirements
------------

None.


Role Variables
--------------

Available variables are listed below along with default values (see `defaults\main.yaml`)

Default configuration of this role keeps most of default configuration file created by td-agent-bit package installation. It only removes default INPUT (cpu monitoring) and OUTPUT (stdout) sections.

You can find more information about each option in the Fluentbit [documentation](https://docs.fluentbit.io/manual/administration/configuring-fluent-bit/configuration-file).


### General configuration variables

* **fluentbit_service_flush_seconds**: Flush time in seconds [default: 5]
* **fluentbit_service_enable_metrics**: Enable metrics HTTP server [default: false]
* **fluentbit_service_log_level**: Set logging verbosity level [default: info]

### Data pipelines configuration

For configuring INPUT, FILTER and OUTPUT section within fluentbit configuration the following variables are used 

* **fluentbit_inputs** : Fluentbit INPUT configuration. List of dictionaries defining log inputs  [default: {}].
* **fluentbit_filters** : Fluentbit FILTER configuration. List of dictionaries defining log filters[default : {}].
* **fluentbit_outputs** : Fluentbit OUTPUT configuration. List of dictionaries defining log outputs [default : {}].

Each dictionary within each list is used to define one `[INPUT]`, `[FILTER]` or `[OUTPUT]` section in fluentbit configuration file. Each configuration section is configured with key/values couples, so the dictionary's keys are used as configuration keys and dictionary's value as values.

For example:

```yml
fluentbit_inputs:
  - name: tail
    tag: auth
    path: "/var/log/auth.log" 
    path_key: log_file 
    DB: "/run/fluent-bit-auth.state"
    Parser: syslog-rfc3164
  - name: tail
    tag: syslog
    path: /var/log/syslog 
    path_key: log_file 
    DB: /run/fluent-bit-syslog.state
    Parser: syslog-rfc3164
```
is translated into:

```
[INPUT]
    name tail
    tag auth
    path /var/log/auth.log
    path_key log_file
    DB /run/fluent-bit-auth.state
    Parser syslog-rfc3164
[INPUT]
    name tail
    tag syslog
    path /var/log/syslog
    path_key log_file
    DB /run/fluent-bit-syslog.state
    Parser syslog-rfc3164

```

Dependencies
------------

None

Example Playbooks
-----------------

```yml
- name: Fluent-bit-installation
  hosts: all
  become: true
  gather_facts: true

  roles:
    - role: ricsanfre.fluentbit
      fluentbit_service_enable_metrics: true
      # Default inputs
      fluentbit_inputs:
        - name: tail
          tag: auth
          path: /var/log/auth.log 
          path_key: log_file 
          DB: /run/fluent-bit-auth.state
          Parser: syslog-rfc3164
        - name: tail
          tag: syslog
          path: /var/log/syslog 
          path_key: log_file 
          DB: /run/fluent-bit-syslog.state
          Parser: syslog-rfc3164
      # Default outputs
      fluentbit_outputs:
        - name: stdout
          match: "*"
```
