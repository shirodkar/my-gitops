Login:
```
UUID=xxxxx
oc login -u admin https://api.cluster-$UUID.dynamic2.redhatworkshops.io:6443 
```
Install GitOps:
```
oc apply -f gitops/install-gitops.yaml
oc get pods -n openshift-gitops --watch
```
Setup Infra:
```
oc apply -f gitops/infra/application-infra.yaml
oc patch console.operator.openshift.io cluster --type=json -p '[{"op":"add","path":"/spec/plugins/-","value":"gitops-plugin"}]'
oc get projects --watch | grep hello-
```
Add OpenBAO (and then manually add the secret values from the OpenBAO UI)
```
oc apply -f gitops/infra/application-openbao.yaml
oc get pods -n openbao --watch
oc exec -n openbao openbao-0 -- sh -c 'bao operator init -key-shares=1 -key-threshold=1'
oc exec -n openbao openbao-0 -- sh -c 'bao operator unseal <unseal_key>'

oc exec -n openbao openbao-0 -- sh -c 'export BAO_TOKEN=s.awWNtuFYCS88hXeLvZWoiy4K && bao secrets enable -version=1 -path=kv kv && bao auth enable kubernetes && bao write auth/kubernetes/config kubernetes_host=https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT && printf "path \"kv/*\" { capabilities = [\"read\",\"list\"] }" | bao policy write eso-policy - && bao write auth/kubernetes/role/eso-role bound_service_account_names=openbao-eso-auth bound_service_account_namespaces=openbao policies=eso-policy ttl=1h && bao write kv/secrets/hello-world/postgres POSTGRESQL_USER="postgres" POSTGRESQL_PASSWORD="postgres" && bao write kv/secrets/hello-world/keystore HTTPS_PASSWORD="password" && bao write kv/secrets/hello-world/quay .dockerconfigjson="{}" && bao write kv/secrets/hello-world/rh-pull-secret .dockerconfigjson="{}" && bao write kv/secrets/hello-world/keystore-file keystore.jks=""'

oc get route -n openbao
```
Deploy Apps
```
oc apply -f gitops/applications/helloworld-ear/application-of-apps.yaml
oc apply -f gitops/applications/honeybees-ear/application-of-apps.yaml
oc get pods -n hello-world-deploy --watch
```
Test Apps
```
oc get route -n hello-world-deploy
oc get route -n honey-bees-deploy
```
