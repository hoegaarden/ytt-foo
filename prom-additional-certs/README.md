# Prometheus with additional certs

Usecase: You want to add an additional scrape target and want to use a client
cert to authenticate with.

The extension does not allow to provide additional certificates to be mounted.
However, we can use an additional templating step to add those certificates.
After that, the cert from a secret is available in the prometheus container and
can be used, e.g. as a scrape config in prometheus' data values:

```yaml
  ...
        - job_name: federation
          static_configs:
          - targets: [ "prom-w1" ]
            labels: { "cluster": "prom-w1" }
          scheme: https
          metrics_path: /federate
          tls_config:
            ca_file: /etc/prometheus/addCerts/prom-w1-client-tls/ca.crt
            cert_file: /etc/prometheus/addCerts/prom-w1-client-tls/tls.crt
            key_file: /etc/prometheus/addCerts/prom-w1-client-tls/tls.key
            server_name: prom-w1
          honor_labels: true
          params:
            'match[]':
            - '{__name__!=""}'
  ...
```

The additional templating step for the prometheus extension can be something like this:

```diff
--- prometheus-extension.yaml   2021-10-25 08:19:17.072338026 -0700
+++ prometheus-extension-additional-certs.yaml  2021-10-25 08:21:04.212069118 -0700
@@ -23,6 +23,74 @@
           pathsFrom:
             - secretRef:
                 name: prometheus-data-values
+    - ytt:
+        ignoreUnknownComments: true
+        inline:
+          paths:
+            overlay.yaml: |
+              #@ load("@ytt:overlay", "overlay")
+              #@ load("@ytt:data", "data")
+              #@ load("@ytt:struct", "struct")
+
+              #@ addCerts = []
+              #@ for cert in data.values.additionalCerts:
+              #@   addCerts.append(struct.decode(cert))
+              #@ end
+
+              #@ matchPromDeploy = overlay.subset({
+              #@   "metadata":{
+              #@     "name":"prometheus-server",
+              #@     "labels":{
+              #@       "component":"server",
+              #@       "app":"prometheus"
+              #@     }
+              #@   },
+              #@   "kind":"Deployment"
+              #@ })
+              #@overlay/match by=matchPromDeploy
+              ---
+              spec:
+                template:
+                  spec:
+                    containers:
+                    #@overlay/match by="name"
+                    - name: prometheus-server
+                      volumeMounts:
+                      #@ for cert in addCerts:
+                      #@overlay/append
+                      - name:      #@ cert.get("volumeName")
+                        mountPath: #@ cert.get("mountPath")
+                        readOnly:  true
+                      #@ end
+                    volumes:
+                    #@ for cert in addCerts:
+                    #@overlay/append
+                    - name: #@ cert.get("volumeName")
+                      secret:
+                        items:
+                        #@ for item in cert.get("items", ["tls.crt", "ca.crt"]):
+                        - path: #@ item
+                          key:  #@ item
+                        #@ end
+                        optional:    #@ cert.get("optional", False)
+                        secretName:  #@ cert.get("secretName")
+                        defaultMode: #@ cert.get("defaultMode", 0o440)
+                    #@ end
+            values.yaml: |
+              #@data/values
+              ---
+              additionalCerts:
+              - secretName: non-existent-secret
+                mountPath:  /etc/prometheus/addCerts/nope
+                volumeName: nope
+                optional:   true
+              - secretName: grafana-tls
+                mountPath:  /etc/prometheus/addCerts/grafana-cert
+                volumeName: grafana-cert
+                items:      [ "tls.crt"]
+              - secretName: grafana-ca-key-pair
+                mountPath:  /etc/prometheus/addCerts/grafana-ca
+                volumeName: grafana-ca-cert
   deploy:
     - kapp:
         rawOptions: ["--wait-timeout=5m"]
```

Note:
- The data values are inlined, but can be pulled from a `Secret` or a `ConfigMap` and by that externalized
- The secrets in the above example are just examples; it probably won't make too much sense to actually use the grafana certs and CA
