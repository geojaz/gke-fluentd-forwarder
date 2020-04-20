# README

## Use-case

Enterprise GKE client wants to "forward" GKE logs from project A to a secondary project B in the normal SD logging screen in GCP console.

Scenario 1: Client needs a birds-eye view of logs... perhaps they have multiple non-prod GCP projects, each having a GKE cluster(s), but they want to be able to aggregate those logs in a single non-prod view via SD.

Scenario 2: Client has configured their GKE clusters to be "multi-tenant". For this, let's assume this means client has several teams that make use of same GKE cluster. They give each team control over 1 or more namespaces and would like to make logs available in SD, but maintain team level access controls.

This approach is potentially useful for both scenarios.

## Details

### TODO: Update for Workload Identity

* Create a service account to be used by the custom fluentd instance that has appropriate permission on your logging project. \
( i think this may be all that's required)
  - Logs writer

* Create a keyfile for that service account
* Create the namespace for log-forwarding `kubectl apply -f deploy/base/namespace.yaml`
* Create a kubernetes secret in your GKE cluster `kubectl create secret generic logging-creds --from-file key.json=$(Location of the keyfile created above)`

* The files in `deploy/base` were copied essentially wholesale from https://github.com/GoogleCloudPlatform/kubernetes-engine-customize-fluentd . I will point out the deviation from this model.

* Update `deploy/overlays/fluentd.configmap.yaml`... This needs to be kustomized, but isn't right now. I pulled it into overlays because my use-case deviates from the base here.

The problem statement here is that the base fluentd doesn't specify the project_id for these sections: `<match {stderr,stdout}>`, `<match **>`
Any section that specifies `@type google_cloud` will need to add the line `project_id $(logging-project)`.
```
      project_id munchie-logs
```

* Deploy the configmap: `kubectl apply -f deploy/overlays/fluentd.configmap.yaml`
* Update the daemonset:

`deploy/base/fluentd.daemonset.yaml` has been updated in two places:

We added an environment variable to the container to configure the `GOOGLE_APPLICATION_CREDENTIALS` which forces the fluentd daemonset to authn as the service account that we set up in a couple steps earlier.

```
        - name: GOOGLE_APPLICATION_CREDENTIALS
          value: /credentials/key.json
```

This envvar is useless without mounting our keyfile secret to this Pod, so we also updated the volumes/volumeMounts:

```
        - mountPath: /credentials
          name: logging-creds
...
      - name: logging-creds
        secret:
          secretName: logging-creds
```

* Deploy the daemonset: `kubectl apply -f deploy/base/fluentd.daemonset.yaml`


## Multitenant setup

Update the `overlays/munchie-spin-labs/fluentd.configmap.yaml`. You'll need to create filters:
```
      <match kubernetes.**{namespace}**>
                @type copy
                <store>
                    @type google_cloud
                    project_id {project}
                    buffer_type file
                    buffer_path /var/log/fluentd-buffers/{namespace}.buffer  # unsure if you need a buffer per namespace
...
```
