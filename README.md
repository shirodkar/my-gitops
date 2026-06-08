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
oc exec -n openbao openbao-0 -- sh -c 'bao operator init -key-shares=1 -key-threshold=1'
oc exec -n openbao openbao-0 -- sh -c 'bao operator unseal <token>'
oc exec -n openbao openbao-0 -- sh -c 'export BAO_TOKEN=<root_token> && bao secrets enable kv && bao auth enable kubernetes && bao write auth/kubernetes/config kubernetes_host=https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT && bao policy write eso-policy - <<EOF
path "kv/data/*" {       
  capabilities = ["read"] 
}                     
path "kv/metadata/*" {   
  capabilities = ["list"] 
}   
EOF && bao write auth/kubernetes/role/eso-role bound_service_account_names=openbao-eso-auth bound_service_account_namespaces=openbao policies=eso-policy ttl=1h && bao write kv/secrets/hello-world/postgres data="{\"POSTGRESQL_PASSWORD\":\"dummy\",\"POSTGRESQL_USER\":\"dummy\"}" &&
bao write kv/secrets/hello-world/keystore data="{\"HTTPS_PASSWORD\":\"dummy\"}" &&
bao write kv/secrets/hello-world/quay data="{\".dockerconfigjson\":\"{\\\"auths\\\":{}}\"}" &&
bao write kv/secrets/hello-world/rh-pull-secret data="{\".dockerconfigjson\":\"{\\\"auths\\\":{}}\"}"'
```

Add Secrets (optional - requires secrets manifests locally):
```
oc apply -f manifests/applications/helloworld-ear/s2i/secrets.yaml -n hello-world-s2i
oc apply -f manifests/applications/helloworld-ear/deploy/secrets.yaml -n hello-world-deploy
oc apply -f manifests/applications/honeybees-ear/s2i/secrets.yaml -n honey-bees-s2i
oc apply -f manifests/applications/honeybees-ear/deploy/secrets.yaml -n honey-bees-deploy
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
