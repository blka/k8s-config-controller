# Configuration controller for Kubernetes

**This is a sidecar application for Kubernetes that unpacks a configmap whenever it changes, and sends a signal to reload an application**.

Under Kubernetes, configmaps mounted into pods generally require an application to be redeployed in order for the application to pick up changes.

This is, of course, often desirable when the application has a release-oriented lifecycle. However, for long-running infrastructure-oriented apps — some examples include Prometheus, Nginx, PostgreSQL and HAProxy — the lifecycle is not released-oriented, but configuration-oriented. For example, adding a new alert to Prometheus should not require a redeploy.

This controller solves this problem by monitoring the configmap for changes and populating a folder with its contents, then reloading the application.

## Operation

The controller will:

* **Unpack the configmap to a directory of your choice**. All files referenced in the configmap are copied relative to a single root directory. Files can be relative; intermediate directories will be automatically created.
* **Signal the application.** This is done via HTTP.
* **Watch for future changes to the configmap.** You should use `kubectl edit` or `kubectl replace` to change the configmap.

Note that there are two requirements:

* The application must be reloadable via HTTP(S).
* The configuration volume must be shared between the sidecar controller and the application's container.
* The configmap is not _mounted_ using `volumeMounts`. The controller takes care of the syncing.

## Environment variables

| **Variable** | **Default** | **Description** |
|------------|-----------|---------------|
| `CONFIG_CONTROLLER_CONFIGROOT` | _(none)_ | The root directory relative to which configmap files are unpacked. |
| `CONFIG_CONTROLLER_CONFIGMAP` | _(none)_ | Name of configmap, e.g. `production/postgresql`. The namespace defaults to `default`. |
| `CONFIG_CONTROLLER_URL` | _(none)_ | URL to issue reload request with |
| `CONFIG_CONTROLLER_METHOD` | `POST` | HTTP method to use. |

[Consult the readme](./README.md) for other arguments that can be specified (if overriding `cmd` or `args`). Among other things, a custom kubeconfig can be specified.

## Example pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: prometheus
  namespace: ops
  labels:
    app: prometheus
spec:
  containers:
  - name: prometheus
    image: quay.io/prometheus/prometheus:latest
    imagePullPolicy: Always
    ports:
    - containerPort: 9090
      protocol: TCP
    livenessProbe:
      httpGet:
        path: /metrics
        port: 9090
        scheme: HTTP
      initialDelaySeconds: 60
      timeoutSeconds: 5
      successThreshold: 1
      failureThreshold: 5
    volumeMounts:
    - name: configs
      mountPath: /mnt/prometheus-config
    - name: data
      mountPath: /mnt/prometheus-data
    args:
    - "-config.file=/mnt/prometheus-config/prometheus.yml"
    - "-storage.local.path=/mnt/prometheus-data"
    - "-web.console.libraries=/etc/prometheus/console_libraries"
    - "-web.console.templates=/etc/prometheus/consoles"
    - "-alertmanager.url=http://alertmanager/"
  - name: config-controller
    image: atombender/k8s-config-controller:latest
    env:
    - name: CONFIG_CONTROLLER_CONFIGROOT
      value: /mnt/prometheus-config
    - name: CONFIG_CONTROLLER_CONFIGMAP
      value: ops/prometheus
    - name: CONFIG_CONTROLLER_URL
      value: http://localhost:9090/-/reload
    - name: CONFIG_CONTROLLER_METHOD
      value: POST
    volumeMounts:
    - name: configs
      mountPath: /mnt/prometheus-config
  volumes:
  # Obviously you will want to use persistent volumes here
  - name: configs
    emptyDir:
  - name: data
    emptyDir:
```

This will obviously require a configmap named `prometheus` in the `ops` namespace.

## Limitations

Does not expose a health check API. We should add this so that if the reload request fails, we can reject the health check, which will cause Kubernetes to kill our pod and do a hard restart of it.
