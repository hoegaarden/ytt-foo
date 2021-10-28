# Prometheus with client cert validation

Usecase: You want to expose a prometheus server, but only want to grant clients
access if they provide a client certificate.

First, enable ingress for prometheus in its data-values:

```yaml
monitoring:
  ingress:
    enabled: true
    virtual_host_fqdn: ...
    ...
```

Secondly, apply another templating step to the prometheus extension to patch its
httpproxy:

```diff
--- prometheus-extension.yaml   2021-10-25 08:19:17.072338026 -0700
+++ prometheus-extension-client-cert.yaml       2021-10-25 09:37:50.312856031 -0700
@@ -23,6 +23,37 @@
           pathsFrom:
             - secretRef:
                 name: prometheus-data-values
+  - ytt:
+      ignoreUnknownComments: true
+      inline:
+        paths:
+          overlay.yaml: |
+            #@ load("@ytt:overlay", "overlay")
+            #@ load("@ytt:data", "data")
+
+            #@ matchPromHTTPProxy = overlay.subset({
+            #@   "metadata":{
+            #@     "name":"prometheus-httpproxy",
+            #@     "labels":{
+            #@       "app":"prometheus"
+            #@     }
+            #@   },
+            #@   "kind":"HTTPProxy"
+            #@ })
+
+            #@ if/end hasattr(data.values, "clientCertCASecret") and data.values.clientCertCASecret:
+            #@overlay/match by=matchPromHTTPProxy
+            ---
+            spec:
+              virtualhost:
+                tls:
+                  #@overlay/match missing_ok=True
+                  clientValidation:
+                    caSecret: #@ data.values.clientCertCASecret
+          values.yaml: |
+            #@data/values
+            ---
+            clientCertCASecret: prom-client-cert-ca
   deploy:
     - kapp:
         rawOptions: ["--wait-timeout=5m"]
```
