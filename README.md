# PoC for Helm v3 to show that Secrets don't always update

This is a PoC for Helm v3 (currently using 3.11.1 and kubectl 1.27.1) that shows how Secrets
aren't updated when entries are removed from them.

Issue opened at: https://github.com/helm/helm/issues/12087

To reproduce, clone this repo, install it in a namespace, check what variables are exposed to
the pod:
```console
$ git clone https://github.com/bebosudo/helm-secrets-not-updated.git
$ cd helm-secrets-not-updated
$ helm upgrade --install --namespace test-secrets-dont-update --create-namespace releasename .
```

Wait for the deployment to finish, then check what variable is defined inside the pod:
```console
$ kubectl -n test-secrets-dont-update get pod
$ kubectl -n test-secrets-dont-update exec deploy/releasename-hello -- env |grep my
myvalue=Hello World
```

Now edit the `secret.yaml` file and uncomment the last line with the entry `myothervar`, upgrade the
installation and check that the new env var is added successfully:
```console
$ vi templates/secret.yaml
$ helm upgrade --install --namespace test-secrets-dont-update --create-namespace releasename .
$ kubectl -n test-secrets-dont-update exec deploy/releasename-hello -- env |grep my
myvalue=Hello World
myothervar=hi again
```

Now reset the `secret.yaml` file, upgrade the installation, and see how the env var persists in
the pod and in the Secret object:
```console
$ git restore templates/secret.yaml
$ helm upgrade --install --namespace test-secrets-dont-update --create-namespace releasename .
$ kubectl -n test-secrets-dont-update exec deploy/releasename-hello -- env |grep my
myothervar=hi again
myvalue=Hello World

$ kubectl -n test-secrets-dont-update get secret hello-secret -o json | jq '.data | map_values(@base64d)'
{
  "myothervar": "hi again",
  "myvalue": "Hello World"
}
```

Notice how Helm didn't detect that the secret was changed on disk, and how this didn't trigger an
update of the checksum in the annotation inside the deployment.

I believe this is due to a special merge that is applied to Secrets, so entries only get added and
never deleted? I tried the same with a ConfigMap, but that works fine.

```console
$ kubectl -n test-secrets-dont-update exec deploy/releasename-hello -- env |grep your
yourvalue=Hello World

$ vi templates/configmap.yaml  # uncomment 'yourothervar'
$ helm upgrade --install --namespace test-secrets-dont-update --create-namespace releasename .
$ kubectl -n test-secrets-dont-update exec deploy/releasename-hello -- env |grep your
yourothervar=hi again
yourvalue=Hello World

$ git restore templates/configmap.yaml
$ helm upgrade --install --namespace test-secrets-dont-update --create-namespace releasename .
$ kubectl -n test-secrets-dont-update exec deploy/releasename-hello -- env |grep your
yourvalue=Hello World
```
