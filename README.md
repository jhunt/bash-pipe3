Advanced Pipelining in Bash
===========================

This code explores the concept of injecting information into a
command pipeline by way of auxiliary file descriptors.  It's a bit
of an odd situation, so here's the "why":

SHIELD is introducing encryption of backups into its wheelhouse.
Currently, the mechanism for driving the SHIELD plugin pipeline is
currently implemented in bash, and uses the environment for
passing information like store and target endpoint configurations,
what plugins to compose, the operation to perform, etc.

Environment variables leak information.  Before, we accepted this
as a necessity, and leaked credentials embedded in target / store
endpoint JSON configurations.  With the introduction of
encryption, we need to tighten up this security.  This means we
cannot pass the encryption initialization vectors and keys via the
environment - too many (unrelated) things would have access,
including:

  - target plugins
  - the compression program (bzip2 / bunzip2)
  - the storage plugin
  - /proc/$pid/environ

For this reason, we want to inject the iv+key into the pipeline at
just the right moment, and we are going to try to do it via an
auxiliary input file descriptor.

Our pipeline looks like this (for backup ops):

```
             +----------+        generates the raw backup archive data
(stderr) <-- |  target  |        and streams it to standard output.
             +----------+
                  |
             +----------+        compresses the raw backup archive data
(stderr) <-- | compress |        so that it takes less space in the
             +----------+        storage system
                  |
             +----------+        encrypts the compressed backup archive
(stderr) <-- | encrypt  |        data to secure the backups
             +----------+
                  |
             +----------+        places the encrypted + compressed
(stderr) <-- |  store   |        backup archive data into the remote
             +----------+        storage system
```

Only box 3, `encrypt` needs to know about the initialization
vector and encryption key, so we want to inject our auxiliary
input file descriptor ("fd 3") there.  Our pipeline now looks like
this:

```
<iv+key> --------------------.
                             |
             +----------+    |   generates the raw backup archive data
(stderr) <-- |  target  |    |   and streams it to standard output.
             +----------+    |
                  |          |
             +----------+    |   compresses the raw backup archive data
(stderr) <-- | compress |    |   so that it takes less space in the
             +----------+    |   storage system
                  |          |
             +----------+    |   encrypts the compressed backup archive
(stderr) <-- | encrypt  | <--'   data to secure the backups
             +----------+
                  |
             +----------+        places the encrypted + compressed
(stderr) <-- |  store   |        backup archive data into the remote
             +----------+        storage system
```

To play with the example implementation:

```
./RUN 3< <(echo "some init vector"; echo "some encryption key")
```
