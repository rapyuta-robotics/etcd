# etcdutl

`etcdutl` is a command line administration utility for [etcd][etcd].

It's designed to operate directly on etcd data files.
For operations over a network, please use `etcdctl`.

### DEFRAG [options]

DEFRAG directly defragments an etcd data directory while etcd is not running.
When an etcd member reclaims storage space from deleted and compacted keys, the space is kept in a free list and the database file remains the same size. By defragmenting the database, the etcd member releases this free space back to the file system.

In order to defrag a live etcd instances over the network, please use `etcdctl defrag` instead.

#### Options

- data-dir -- Optional. If present, defragments a data directory not in use by etcd.

#### Output

Exit status '0' when the process was successful.

#### Example

To defragment a data directory directly, use the `--data-dir` flag:

``` bash
# Defragment while etcd is not running
./etcdutl defrag --data-dir default.etcd
# success (exit status 0)
# Error: cannot open database at default.etcd/member/snap/db
```

#### Remarks

DEFRAG returns a zero exit code only if it succeeded in defragmenting all given endpoints.

### SNAPSHOT RESTORE [options] \<filename\>

SNAPSHOT RESTORE creates an etcd data directory for an etcd cluster member from a backend database snapshot and a new cluster configuration. Restoring the snapshot into each member for a new cluster configuration will initialize a new etcd cluster preloaded by the snapshot data.

#### Options

The snapshot restore options closely resemble to those used in the `etcd` command for defining a cluster.

- data-dir -- Path to the data directory. Uses \<name\>.etcd if none given.

- wal-dir -- Path to the WAL directory. Uses data directory if none given.

- initial-cluster -- The initial cluster configuration for the restored etcd cluster.

- initial-cluster-token -- Initial cluster token for the restored etcd cluster.

- initial-advertise-peer-urls -- List of peer URLs for the member being restored.

- name -- Human-readable name for the etcd cluster member being restored.

- skip-hash-check -- Ignore snapshot integrity hash value (required if copied from data directory)

- bump-revision -- How much to increase the latest revision after restore

- mark-compacted -- Mark the latest revision after restore as the point of scheduled compaction (required if --bump-revision > 0, disallowed otherwise)

#### Output

A new etcd data directory initialized with the snapshot.

#### Example

Save a snapshot, restore into a new 3 node cluster, and start the cluster:
```
# save snapshot
./etcdctl snapshot save snapshot.db

# restore members
./etcdutl snapshot restore snapshot.db --initial-cluster-token etcd-cluster-1 --initial-advertise-peer-urls http://127.0.0.1:12380  --name sshot1 --initial-cluster 'sshot1=http://127.0.0.1:12380,sshot2=http://127.0.0.1:22380,sshot3=http://127.0.0.1:32380'
./etcdutl snapshot restore snapshot.db --initial-cluster-token etcd-cluster-1 --initial-advertise-peer-urls http://127.0.0.1:22380  --name sshot2 --initial-cluster 'sshot1=http://127.0.0.1:12380,sshot2=http://127.0.0.1:22380,sshot3=http://127.0.0.1:32380'
./etcdutl snapshot restore snapshot.db --initial-cluster-token etcd-cluster-1 --initial-advertise-peer-urls http://127.0.0.1:32380  --name sshot3 --initial-cluster 'sshot1=http://127.0.0.1:12380,sshot2=http://127.0.0.1:22380,sshot3=http://127.0.0.1:32380'

# launch members
./etcd --name sshot1 --listen-client-urls http://127.0.0.1:2379 --advertise-client-urls http://127.0.0.1:2379 --listen-peer-urls http://127.0.0.1:12380 &
./etcd --name sshot2 --listen-client-urls http://127.0.0.1:22379 --advertise-client-urls http://127.0.0.1:22379 --listen-peer-urls http://127.0.0.1:22380 &
./etcd --name sshot3 --listen-client-urls http://127.0.0.1:32379 --advertise-client-urls http://127.0.0.1:32379 --listen-peer-urls http://127.0.0.1:32380 &
```

### SNAPSHOT STATUS \<filename\>

SNAPSHOT STATUS lists information about a given backend database snapshot file.

#### Output

##### Simple format

Prints a humanized table of the database hash, revision, total keys, and size.

##### JSON format

Prints a line of JSON encoding the database hash, revision, total keys, and size.

#### Examples
```bash
./etcdutl snapshot status file.db
# cf1550fb, 3, 3, 25 kB
```

```bash
./etcdutl --write-out=json snapshot status file.db
# {"hash":3474280699,"revision":3,"totalKey":3,"totalSize":24576}
```

```bash
./etcdutl --write-out=table snapshot status file.db
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| cf1550fb |        3 |          3 | 25 kB      |
+----------+----------+------------+------------+
```

### HASHKV [options] \<filename\>

HASHKV prints hash of keys and values up to given revision.

#### Options

- rev -- Revision number. Default is 0 which means the latest revision.

#### Output

##### Simple format

Prints a humanized table of the KV hash, hash revision and compact revision.

##### JSON format

Prints a line of JSON encoding the KV hash, hash revision and compact revision.

#### Examples
```bash
./etcdutl hashkv file.db
# 35c86e9b, 214, 150
```

```bash
./etcdutl --write-out=json hashkv file.db
# {"hash":902327963,"hashRevision":214,"compactRevision":150}
```

```bash
./etcdutl --write-out=table hashkv file.db
+----------+---------------+------------------+
|   HASH   | HASH REVISION | COMPACT REVISION |
+----------+---------------+------------------+
| 35c86e9b |           214 |              150 |
+----------+---------------+------------------+
```

### VERSION

Prints the version of etcdutl.

#### Output

Prints etcd version and API version.

#### Examples


```bash
./etcdutl version
# etcdutl version: 3.5.0
# API version: 3.1
```

### LIST-BUCKET [options] \<data dir or db file path\>

`list-bucket` prints all bucket names.

#### Flags

- timeout -- Time to wait to obtain a file lock on db file, 0 to block indefinitely.

##### Examples for LIST-BUCKET

```bash

$ ./etcdutl list-bucket ~/tmp/etcd/default.etcd/member/snap/db
alarm
auth
authRoles
authUsers
cluster
key
lease
members
members_removed
meta
```

### ITERATE-BUCKET [options] \<data dir or db file path\> \<bucket name\>

`iterate-bucket` lists key-value pairs in a given bucket in reverse order.

#### Flags for ITERATE-BUCKET

- timeout -- Time to wait to obtain a file lock on db file, 0 to block indefinitely.
- limit -- Max number of key-value pairs to iterate (0 to iterate all).
- decode -- true to decode Protocol Buffer encoded data.

##### Examples for ITERATE-BUCKET

```bash

# with `--decode` option
$ ./etcdutl iterate-bucket ~/tmp/etcd/default.etcd/member/snap/db key --decode
rev={Revision:{Main:4 Sub:0} tombstone:false}, value=[key "k1" | val "v3" | created 2 | mod 4 | ver 3]
rev={Revision:{Main:3 Sub:0} tombstone:false}, value=[key "k1" | val "v2" | created 2 | mod 3 | ver 2]
rev={Revision:{Main:2 Sub:0} tombstone:false}, value=[key "k1" | val "v1" | created 2 | mod 2 | ver 1]

# without `--decode` option
$ ./etcdutl iterate-bucket ~/tmp/etcd/default.etcd/member/snap/db key
key="\x00\x00\x00\x00\x00\x00\x00\x04_\x00\x00\x00\x00\x00\x00\x00\x00", value="\n\x02k1\x10\x02\x18\x04 \x03*\x02v3"
key="\x00\x00\x00\x00\x00\x00\x00\x03_\x00\x00\x00\x00\x00\x00\x00\x00", value="\n\x02k1\x10\x02\x18\x03 \x02*\x02v2"
key="\x00\x00\x00\x00\x00\x00\x00\x02_\x00\x00\x00\x00\x00\x00\x00\x00", value="\n\x02k1\x10\x02\x18\x02 \x01*\x02v1"
```

### HASH [options] \<data dir or db file path\>

`hash` prints the hash of the db file.

#### Flags for HASH

- timeout -- Time to wait to obtain a file lock on db file, 0 to block indefinitely.

##### Examples for HASH

```bash

$ ./etcdutl hash ~/tmp/etcd/default.etcd/member/snap/db
db path: /Users/wachao/tmp/etcd/default.etcd/member/snap/db
Hash: 4031086527
```

## Exit codes

For all commands, a successful execution returns a zero exit code. All failures will return non-zero exit codes.

## Output formats

All commands accept an output format by setting `-w` or `--write-out`. All commands default to the "simple" output format, which is meant to be human-readable. The simple format is listed in each command's `Output` description since it is customized for each command. If a command has a corresponding RPC, it will respect all output formats.

If a command fails, returning a non-zero exit code, an error string will be written to standard error regardless of output format.

### Simple

A format meant to be easy to parse and human-readable. Specific to each command.

### JSON

The JSON encoding of the command's [RPC response][etcdrpc]. Since etcd's RPCs use byte strings, the JSON output will encode keys and values in base64.

Some commands without an RPC also support JSON; see the command's `Output` description.

### Protobuf

The protobuf encoding of the command's [RPC response][etcdrpc]. If an RPC is streaming, the stream messages will be concatenated. If an RPC is not given for a command, the protobuf output is not defined.

### Fields

An output format similar to JSON but meant to parse with coreutils. For an integer field named `Field`, it writes a line in the format `"Field" : %d` where `%d` is go's integer formatting. For byte array fields, it writes `"Field" : %q` where `%q` is go's quoted string formatting (e.g., `[]byte{'a', '\n'}` is written as `"a\n"`).

## Compatibility Support

etcdutl is still in its early stage. We try out best to ensure fully compatible releases, however we might break compatibility to fix bugs or improve commands. If we intend to release a version of etcdutl with backward incompatibilities, we will provide notice prior to release and have instructions on how to upgrade.

### Input Compatibility

Input includes the command name, its flags, and its arguments. We ensure backward compatibility of the input of normal commands in non-interactive mode.

### Output Compatibility
Currently, we do not ensure backward compatibility of utility commands.

### TODO: compatibility with etcd server

[etcd]: https://github.com/coreos/etcd
[READMEv2]: READMEv2.md
[v2key]: ../store/node_extern.go#L28-L37
[v3key]: ../api/mvccpb/kv.proto#L12-L29
[etcdrpc]: ../api/etcdserverpb/rpc.proto
[storagerpc]: ../api/mvccpb/kv.proto
