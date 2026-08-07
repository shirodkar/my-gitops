Login:
Log in to OpenShift as a cluster admin.

Install GitOps:
```
oc apply -f gitops/install-gitops.yaml
oc get pods -n openshift-gitops --watch
```
Setup Infra [uses Helm]:
```
oc apply -f gitops/infra/application-infra.yaml
oc patch console.operator.openshift.io cluster --type=json -p '[{"op":"add","path":"/spec/plugins/-","value":"gitops-plugin"}]'
oc get projects --watch | grep hello-
```
Add OpenBAO (and then manually add the secret values from the OpenBAO UI) [uses Plain Manifests]
```
oc apply -f gitops/infra/application-openbao.yaml
oc get pods -n openbao --watch
oc exec -n openbao openbao-0 -- sh -c 'bao operator init -key-shares=1 -key-threshold=1'
oc exec -n openbao openbao-0 -- sh -c 'bao operator unseal <unseal_key>'

oc exec -n openbao openbao-0 -- sh -c 'export BAO_TOKEN=<root_token> && bao secrets enable -version=1 -path=kv kv && bao auth enable kubernetes && bao write auth/kubernetes/config kubernetes_host=https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT && printf "path \"kv/*\" { capabilities = [\"read\",\"list\"] }" | bao policy write eso-policy - && bao write auth/kubernetes/role/eso-role bound_service_account_names=openbao-eso-auth bound_service_account_namespaces=openbao policies=eso-policy ttl=1h && bao write kv/secrets/hello-world/postgres POSTGRESQL_USER="postgres" POSTGRESQL_PASSWORD="postgres" && bao write kv/secrets/hello-world/keystore HTTPS_PASSWORD="password" && bao write kv/secrets/hello-world/quay .dockerconfigjson="{}" && bao write kv/secrets/hello-world/rh-pull-secret .dockerconfigjson="{}" && bao write kv/secrets/hello-world/keystore-file keystore.jks=""'

oc get route -n openbao
```
Deploy Apps [uses Kustomize]
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

MTA
```
oc apply -f gitops/infra/application-mta.yaml 
oc get pods -n openshift-mta -w

CLUSTER_URL_SUFFIX=oc whoami -c | sed -E 's|[^/]+/api-([^:]+):[0-9]+/.*|\1|'
HUB="https://mta-openshift-mta.apps.$CLUSTER_URL_SUFFIX/hub"

EAP_TAG_ID=$(curl -sk "$HUB/tags" | jq '[.[] | select(.name=="EAP" and .category.name=="Runtime")][0].id')
if [ "$EAP_TAG_ID" = "null" ] || [ -z "$EAP_TAG_ID" ]; then
  echo "ERROR: EAP tag not found"; exit 1
fi
echo "EAP_TAG_ID=$EAP_TAG_ID"

AP_ID=$(curl -sk -X POST "$HUB/analysis/profiles" \
  -H "Content-Type: application/json" \
  -d "$(cat <<'EOF'
{
  "name": "EAP7 to EAP8",
  "mode": {"withDeps": true},
  "scope": {"withKnownLibs": true, "packages": {"included": [], "excluded": []}},
  "rules": {
    "targets": [
      {"id": 1, "selection": "konveyor.io/target=eap8"},
      {"id": 6, "selection": "konveyor.io/target=openjdk17"},
      {"id": 8},
      {"id": 9}
    ],
    "labels": {"included": [], "excluded": []},
    "repository": {
      "kind": "git",
      "url": "https://github.com/shirodkar/mammoth-ear.git",
      "branch": "main",
      "path": "/rules/"
    }
  }
}
EOF
)" | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
echo "AP_ID=$AP_ID"

curl -sk -X POST "$HUB/applications" \
  -H "Content-Type: application/json" \
  -d "{
    \"name\": \"Mammoth\",
    \"repository\": {\"kind\": \"git\", \"url\": \"https://github.com/shirodkar/mammoth-ear.git\"},
    \"tags\": [{\"id\": $EAP_TAG_ID}]
  }"

ARCH_ID=$(curl -sk -X POST "$HUB/archetypes" \
  -H "Content-Type: application/json" \
  -d "{
    \"name\": \"eap7\",
    \"criteria\": [{\"id\": $EAP_TAG_ID}]
  }" | jq '.id')
echo "ARCH_ID=$ARCH_ID"

curl -sk -X PUT "$HUB/archetypes/$ARCH_ID" \
  -H "Content-Type: application/json" \
  -d "{
    \"name\": \"eap7\",
    \"criteria\": [{\"id\": $EAP_TAG_ID}],
    \"profiles\": [{
      \"name\": \"eap8\",
      \"analysisProfile\": {\"id\": $AP_ID}
    }]
  }"
```

Dev Spaces
```
oc apply -f gitops/infra/application-devspaces.yaml 
oc get pods -n openshift-devspaces -w
