# Grafana with LDAP

The TKG 1.3.1 extension has a bug, where it does not really use the LDAP config
for grafana, even if configured in the data values.

This patch fixes that.

```diff
--- grafana-extension.yaml      2021-10-28 02:34:49.658690788 -0700
+++ grafana-extension-ldap.yaml 2021-10-28 02:37:37.646323574 -0700
@@ -23,6 +23,64 @@
           pathsFrom:
             - secretRef:
                 name: grafana-data-values
+    - ytt:
+        inline:
+          paths:
+            overlay.yaml: |
+              #@ load("@ytt:overlay", "overlay")
+
+              #@ matchGrafanaDeploy = overlay.subset({
+              #@   "metadata": {
+              #@     "name": "grafana",
+              #@     "labels": {
+              #@        "app.kubernetes.io/name": "grafana"
+              #@     }
+              #@   },
+              #@   "kind": "Deployment"
+              #@ })
+              #@overlay/match by=matchGrafanaDeploy
+              ---
+              spec:
+                template:
+                  spec:
+                    containers:
+                    #@overlay/match by="name"
+                    - name: grafana
+                      volumeMounts:
+                      #@overlay/append
+                      - name: ldap-toml
+                        mountPath: /etc/grafana/ldap.toml
+                        readOnly: true
+                        subPath: ldap-toml
+                    volumes:
+                    #@overlay/append
+                    - name: ldap-toml
+                      secret:
+                        secretName: grafana
+                        items: [{path: ldap-toml, key: ldap-toml}]
+              ---
+              #@ def patchGrafanaIni(org, additional):
+              #@   return org + "\n\n" + additional
+              #@ end
+
+              #@ matchGrafanaConfigMap = overlay.subset({
+              #@   "metadata": {
+              #@     "name": "grafana",
+              #@     "labels": {
+              #@        "app.kubernetes.io/name": "grafana"
+              #@     }
+              #@   },
+              #@   "kind": "ConfigMap"
+              #@ })
+              #@overlay/match by=matchGrafanaConfigMap
+              ---
+              data:
+                #@overlay/replace via=patchGrafanaIni
+                grafana.ini: |
+                  [auth.ldap]
+                  enabled = true
+                  config_file = /etc/grafana/ldap.toml
+                  allow_sign_up = true
   deploy:
     - kapp:
         rawOptions: ["--wait-timeout=5m"]
```
