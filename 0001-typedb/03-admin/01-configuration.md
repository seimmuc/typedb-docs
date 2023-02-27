---
pageTitle: Configuration
keywords: typedb, configuration, administration, config, settings
longTailKeywords: TypeDB administration, TypeDB configuration, TypeDB settings, changing settings
summary: TypeDB configuration guide.
toc: false
---

# Configuration

There are two ways to configure TypeDB: 

* command line arguments,
* configuration file.

## Command line arguments

Use command line arguments with the command `typedb server` to configure the server upon its launch. The command line 
arguments will override any relevant settings from the config file.

All arguments must: 

* start with the double dash prefix `--`,
* be separated from their value (if any) by:
    * equals sign (`=`),
    * or whitespace (` `).

Some arguments are exclusive to the command line interface:

* `--help`: print out the help menu and exit.
* `--version`: print out the version of the server and exit.
* `--config /path/to/external/typedb-config.yml`: use the specified configuration file.

Additional arguments available are derived from the configuration `yaml` file. Any value from the configuration file 
can be overridden by a command line argument, as shown in the relevant
[section](#configuration-file-options-via-command-line-arguments).

### Getting help

For all available commands via the command line use the `--help` argument to get a reference:

<!-- test-ignore -->
```bash
typedb server --help
```

This will list out how to use command line specific arguments as well as options derived from the configuration file.

## Configuration file

TypeDB accepts a configuration (config) file with a specific YAML format. See the
[sample configuration](#sample-configuration) below.

<div class="note">
[Important]
In order for any change in configuration file to take effect, restart the TypeDB server.
</div>

### The default location of the config file

TypeDB ships with a default configuration file. The location of this file varies based on how TypeDB has been installed.

If downloaded manually, find the configuration file in the `server/conf` directory inside the unzipped folder.

If installed using Homebrew, check the following directory (replace the `{version-number}` variable with number of 
installed version):

<!-- test-ignore -->
```
/usr/local/Cellar/typedb/{version-number}/libexec/server/conf/config.yml
```

If installed using APT:

<!-- test-ignore -->
```
/opt/typedb/core/server/conf/config.yml
```

### Sample configuration

Here is a sample configuration file for TypeDB:

<!-- test-ignore -->
```yaml
server:
  address: 0.0.0.0:1729
storage:
  data: server/data
  database-cache:
    # configure storage-layer data and index cache per database 
    # it is recommended to keep these at equal sizes
    data: 500mb
    index: 500mb
log:
  output:
    stdout: # note: this is a user-defined name
      type: stdout
    file: # note: this is a user-defined name
      type: file
      directory: server/logs
      file-size-cap: 50mb
      archives-size-cap: 1gb
  logger:
    default: # note: the default logger must be defined
      level: warn
      output: [ stdout, file ]
    typedb: # note: this is a user-defined name
      filter: com.vaticle.typedb.core
      level: info
      output: [ stdout, file ]
    storage:
      filter: com.vaticle.typedb.core.database
      level: info # on 'debug' the server will periodically log database storage properties
      output: [ stdout, file ]
  debugger:
    reasoner: # note: this is a user-defined name
      enable: false
      type: reasoner
      output: file
vaticle-factory:
  enable: false
```

### Server configuration

The server section of the configuration file contains network and RPC options.

* `address`: the address to listen for gRPC clients on.

  Use the `0.0.0.0` IP-address to listen to all connections and `localhost` for connections from the local machine.

### Storage configuration

The storage section of the configuration file contains the storage layer options of TypeDB.

* `data`: the location to write user data to. Defaults to within the server distribution under server/data.
* `database-cache`: **per-database** configuration for storage-level caching
    * `data`: cache for often-used data.
    * `index`: cache for data indexes.

<div class="note">
[Important]
For production use, it is recommended that the `server.data` is set to a path outside of the `$TYPEDB_HOME` (directory 
with TypeDB server files). This helps to make the process of upgrading TypeDB easier.
</div>

If the index cache is too small relative to the dataset, we may find severely degraded performance. We recommend 
allocating at least **2%** of a database size equivalent to the index cache. For example, with **100 GB** of 
on-disk data in a database, allocate at least **2 GB** of index cache. Allocating more can improve performance.

Additionally, we recommend the sum of data and index caches equal to about **20%** of the total memory of the server.

### Logging configuration

The log section of the configuration file contains the logging behavior options of TypeDB.

There are three subsections:

* `output`: Define destinations to write logs to. Allowed types are `type: file`, and `type: stdout` in TypeDB.
* `logger`: Set up logging for modules in TypeDB, along with a log level and output targets (referencing outputs by name defined under the outputs section).
* `debugger`: Set up TypeDB-specific debuggers. Right now, the only defined type is `type: reasoner`.

## Configuration file options via command line arguments

Use command line arguments to override any option in the configuration file.

For example, the configuration file sets the server address as:

<!-- test-ignore -->
```yaml
server:
  address: 0.0.0.0:1729
```

If we want to use port 1730 instead of 1729, we can either update the configuration file or override it from the 
command line using the following command:

```bash
typedb server --server.address 0.0.0.0:1730
```

Use the same approach to set a completely new section of the configuration that isn’t present in the file yet. For 
example, to define a new logger subsection to print out all query plans, we could do the following to set the package 
`com.vaticle.typedb.core.traversal` to output on a more verbose level:

<!-- test-ignore -->
```yaml
typedb server  \
  --server.address 0.0.0.0:1730  \
  --log.logger.traversal.filter com.vaticle.typedb.core.traversal  \
  --log.logger.traversal.level debug \
  --log.logger.traversal.output "[ file, stdout ]"
```

## Cluster configuration

Every server in a cluster has its own config file that contains a list of known servers in the cluster. A server in a 
cluster will not accept connections from servers that are not in the list.

<div class="note">
[Note]
Changes to the server configuration require a server restart to take effect.
</div>

### Add or remove cluster's servers

To add or remove a server to/from a cluster:

1. Stop all TypeDB servers in the cluster.
2. Update the configuration files of all (both new and old) TypeDB servers.
3. Start all TypeDB servers of the new cluster.

## Host machine requirements

The minimum host machine configuration for running a single TypeDB database is 4 (v)CPUs, 10 GB memory, with SSD.

The recommended starting configuration is 8 (v)CPUs, 16 GB memory, and SSD. Bulk loading is scaled effectively by 
adding more CPU cores.

The memory breakdown of TypeDB is the following:

* the JVM memory: configurable when booting the server with `JAVAOPTS="-Xmx4g"` typedb server. This gives the JVM 4 GB 
  of memory. Defaults to **25%** of system memory on most machines.
* storage-layer baseline consumption: approximately **2 GB**.
* storage-layer caches: this is about **2x** cache size per database. If the **data and index caches** sum up to **1 GB**, 
  the memory requirement is **2 GB** in working memory.
* memory per CPU: approximately **0.5 GB** additional per (v)CPU under full load.

We can estimate the amount of memory the server will need to run a single database with these factors:

`required memory = JVM memory + 2 GB + (2 × configured db-caches in GB) + (0.5 GB × CPUs)`

For example, on a 4 CPU machine, with the default 1 GB of per-database storage caches, and the JVM using 4 GB of RAM,
the default requirement for memory would be: 4 GB + 2 GB + (2 × 1 GB) + (0.5 GB × 4) = **10 GB**.

Each additional database will consume an additional amount at least equal to the cache requirements (in this example, 
an additional 2 GB of memory for each database).

### Open file limit

To support large data volumes, it is important to check the open file limit the operating system imposes. Some Unix 
distributions default to `1024` open file descriptors. This can be checked with:

<!-- test-ignore -->
```bash
ulimit -n
```

We recommend this is increased to at least `50 000`.