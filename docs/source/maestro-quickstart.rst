Maestro Quick Start
######################

ldms_cluster yaml file
***************************

daemons
=======
This section defines the ldmsd daemon configurations in a given cluster

names
-----
string value of the name of daemons in a config; mult daemons can be created with string formatting
e.x. "sampler-[1-10]" will create 10 daemons ['sampler-1', 'sampler-2', ... ]
ampersand '&' will assign variable name to string in yaml format
e.g. &sampler-daemons "sampler-[1-10]"

hosts
-----
String value that represents the hostnames of the hosts the daemons will be running on
If the amount of hosts listed is not equal to the amount of daemons, maestro will balance the daemons across hosts

maestro_comm
------------
Boolean value that states whether maestro will attempt to communicate/configure the daemon(s)

endpoints
---------
Endpoints by which maestro or other ldmsd's will communicate with daemons
e.g. &sampler-endpoints "node-[1-10]-[10001-10002]" will expand to ["node-1-10001", "node-1-10002", ..., ]; i.e. 2 ports per ld

names
`````
Names for endpoints, current convention is {dmn_name-1-port}
e.g. "node-[1-10]-[10001]"

ports
`````
String of port numbers
e.g. "[10001-10002]"

auth
````
Dictionary of the authentication domain and it's config for these endpoints

name
''''
Arbitrary name for authentication domain. Allows multiple instances of munge, or ovis
e.g. munge1

plugin
''''''
Authentication domain plugin type
<ovis|munge|none>

aggregators
===========
A yaml list of aggregator configuration definitions

daemons
-------
Names of daemons to assign the aggregator configurations

peers
-----
This section defines the ldmsd producers that the aggregator(s) will be adding

endpoints
`````````
The producer endpoints to aggregate sets from

reconnect
`````````
Timeout before aggregator attempts reconnecting to producer

type
````
<active|passive>

updaters
````````
This section is a list of dictionaries defining updater configurations

mode
''''
<pull|auto|push|onchange>
pull - default, updater will schedule set updates according to defined interval and offset values
auto - updater will schedule set updates according to update hint
push - updater will receive an update only when the set source explicitly pushes an update
onchange - updater will get an update whenever the set source ends a transaction or pushes an update

interval
''''''''
Either an integer assumed to be in microseconds or a String value with a designator
s - seconds
ms - milliseconds
us - microseconds
e.g. "1.0s" or "1000000us" or 1000000

offset
''''''

sets
''''
This section allows you to define which producer sets to apply this updater to

regex
.....
This is a regular expression that matches either set instance names, or schema names. ".*" will include all sets
e.g. 'meminfo'

field
.....
<inst|schema>
inst - will match regex to set instance names
schema - will match regex to schema set

samplers
========
The sampler daemon and plugin configuration for sampler daemons

daemons
-------
String value, names of daemons to apply the sampler configuration
e.g. "sampler-[1-10]"

config
------
This section is a list of dictionaries of sampler plugins and their configurations

name
''''
Name of the sampler plugin
e.g. 'meminfo'


Example
=======
.. code-block:: yaml
  daemons:
    - names : &samplers "sampler-[1-10]"
      hosts : &node-hosts "node-[1-10]"
      endpoints :
        - names : &sampler-endpoints "node-[1-10]-[10002]"
          ports : &sampler-ports "[10002]"
          maestro_comm : True
          xprt  : sock
          auth  :
             name : ovis1 # The authentication domain name
             plugin : ovis # The plugin type
             conf : /opt/ovis/maestro/secret.conf

        - names : &sampler-rdma-endpoints "node-[1-10]-10002/rdma"
          ports : *sampler-ports
          maestro_comm : False
          xprt  : rdma
          auth  :
            name : munge1
            plugin : munge

    - names : &l1-agg "l1-aggs-[11-14]"
      hosts : &l1-agg-hosts "node-[11-14]"
      endpoints :
        - names : &l1-agg-endpoints "node-[11-14]-[10101]"
          ports : &agg-ports "[10101]"
          maestro_comm : True
          xprt  : sock
          auth  :
            name : munge1
            plugin : munge

    - names : &l2-agg "l2-agg"
      hosts : &l2-agg-host "node-15"
      endpoints :
        - names : &l2-agg-endpoints "node-[15]"
          ports : "[10104]"
          maestro_comm : True
          xprt  : sock
          auth  :
            name : munge1
            plugin : munge

  aggregators:
    - daemons   : *l1-agg
      peers     :
        - endpoints : *sampler-endpoints
          reconnect : 20s
          type      : active
          updaters  :
            - mode     : pull
              interval : "1.0s"
              offset   : "0ms"
              sets     :
                - regex : .*
                  field : inst

    - daemons   : *l2-agg
      peers     :
        - endpoints : *l1-agg-endpoints
          reconnect : 20s
          type      : active
          updaters  :
            - mode : pull
              interval : "1.0s"
              offset   : "0ms"
              sets     :
                - regex : .*
                  field : inst

  samplers:
    - daemons : *samplers
      config :
        - name        : meminfo # Variables can be specific to plugin
          interval    : "1.0s" # Used when starting the sampler plugin
          offset      : "0ms"
          perm        : "0777"

        - name        : vmstat
          interval    : "1.0s"
          offset      : "0ms"
          perm        : "0777"

        - name        : procstat
          interval    : "1.0s"
          offset      : "0ms"
          perm        : "0777"

  stores:
    - name      : sos-meminfo
      daemons   : *l2-agg
      container : ldms_data
      schema    : meminfo
      flush     : 10s
      plugin :
        name   : store_sos
        config : { path : /DATA }

    - name      : sos-vmstat
      daemons   : *l2-agg
      container : ldms_data
      schema    : vmstat
      flush     : 10s
      plugin :
        name   : store_sos
        config : { path : /DATA }

    - name      : sos-procstat
      daemons   : *l2-agg
      container : ldms_data
      schema    : procstat
      flush     : 10s
      plugin :
        name   : store_sos
        config : { path : /DATA }

    - name : csv
      daemons   : *l2-agg
      container : ldms_data
      schema    : meminfo
      plugin :
        name : store_csv
        config :
          path        : /DATA/csv/
          altheader   : 0
          typeheader  : 1
          create_uid  : 3031
          create_gid  : 3031


This is the end of the code block
