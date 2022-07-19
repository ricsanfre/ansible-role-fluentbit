Ansible Role: Fluentbit
=========

Fluentbit installation and configuration role.


Requirements
------------

None.


Role Variables
--------------

Available variables are listed below along with default values (see `defaults\main.yaml`)

Default configuration of this role keeps most of default configuration file created by fluent-bit package installation. It only removes default INPUT (cpu monitoring) and OUTPUT (stdout) sections.

You can find more information about each option in the Fluentbit [documentation](https://docs.fluentbit.io/manual/administration/configuring-fluent-bit/configuration-file).


### General configuration variables

* **fluentbit_service_flush_seconds**: Flush time in seconds [default: 5]
* **fluentbit_service_enable_metrics**: Enable metrics HTTP server [default: false]
* **fluentbit_service_log_level**: Set logging verbosity level [default: info]

### Data pipelines configuration

For configuring INPUT, FILTER and OUTPUT section within fluentbit configuration the following variables are used 

* **fluentbit_inputs**: Fluentbit INPUT configuration section [default: ""].
* **fluentbit_filters**: Fluentbit FILTER configuration section [default: ""].
* **fluentbit_outputs**: Fluentbit OUTPUT configuration section [default: ""].


Using that variables the config file content can be added using using yaml `|` operator

```yml
fluentbit_inputs: |
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


### Parsers configuration

Default package fluent-bit installation comes with a predefined set of parsers included as part of `parsers.conf` file. Additional configuration files containing parsers can be added to fluent-bit configuration. This role uses `custom_parser.conf` file for keeping additional custom parsers.

Content of this custom parser file can be configured using the variable `fluentbit_custom_parsers`

```yml
fluentbit_custom_parsers: |
  [PARSER]
      Name syslog-rfc3164-nopri
      Format regex
      Regex /^(?<time>[^ ]* {1,2}[^ ]* [^ ]*) (?<host>[^ ]*) (?<ident>[a-zA-Z0-9_\/\.\-]*)(?:\[(?<pid>[0-9]+)\])?(?:[^\:]*\:)? *(?<message>.*)$/
      Time_Key time
      Time_Format %b %d %H:%M:%S
      Time_Keep False
```


### Lua filters and lua script support

Lua Filter allows you to modify the incoming records using custom Lua Scripts. See fluentbit [documentation](https://docs.fluentbit.io/manual/pipeline/filters/lua)

Lua filter can be included in `fluentbit_filters` variable

```yml
fluentbit_filters: |
  [FILTER]
      Name lua
      Match host.*
      script /etc/fluent-bit/adjust_ts.lua
      call local_timestamp_to_UTC
```
Lua scripts used by the filters need to be configured using `fluentbit_lua_scripts` role variable. This variable is a list of dictionaries with 2 fields:
- `name`: name of the lua script
- `content`: script content)

Script content can be populated from a file using ansible [`lookup` plugin](https://docs.ansible.com/ansible/latest/plugins/lookup.html).

```yml
fluentbit_lua_scripts:
  - name: adjust_ts.lua
    content: "{{ lookup('template','templates/adjust_ts.lua') }}"

```
and `templates/adjust_ts.lua` file content:

```lua
function local_timestamp_to_UTC(tag, timestamp, record)
    local utcdate   = os.date("!*t", ts)
    local localdate = os.date("*t", ts)
    localdate.isdst = false -- this is the trick
    utc_time_diff = os.difftime(os.time(localdate), os.time(utcdate))
    return 1, timestamp - utc_time_diff, record
end
```

> NOTE: all lua scripts will be copied to `/etc/fluent-bit` directory

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
      # lua scripts
      fluentbit_lua_scripts:
        - name: adjust_ts.lua
          content: "{{ lookup('template','templates/adjust_ts.lua') }}"
      # inputs
      fluentbit_inputs: |
        [INPUT]
            Name tail
            Tag host.*
            DB /run/fluentbit-state.db
            Path /var/log/auth.log,/var/log/syslog
            Parser syslog-rfc3164-nopri
      # filters
      fluentbit_filters: |
        [FILTER]
            Name lua
            Match host.*
            script /etc/fluent-bit/adjust_ts.lua
            call local_timestamp_to_UTC
      # outputs
      fluentbit_outputs: |
        [OUTPUT]
            Name stdout
            Match *
      # Parsers
      fluentbit_custom_parsers: |
        [PARSER]
            Name syslog-rfc3164-nopri
            Format regex
            Regex /^(?<time>[^ ]* {1,2}[^ ]* [^ ]*) (?<host>[^ ]*) (?<ident>[a-zA-Z0-9_\/\.\-]*)(?:\[(?<pid>[0-9]+)\])?(?:[^\:]*\:)? *(?<message>.*)$/
            Time_Key time
            Time_Format %b %d %H:%M:%S
            Time_Keep False
```
