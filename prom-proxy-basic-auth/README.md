# Prometheus proxy with basic auth

This has been used to slap a proxy in front of TKG 1.3.1 extensions, so that the
prometheus in workload clusters is somewhat protected, namely via HTTP basic auth.

The config of this kapp is inlined, but can be changed to use a data-values
secret, similar to other TKG extensions.
