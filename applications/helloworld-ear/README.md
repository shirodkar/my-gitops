Install GitOps:
```
oc apply -f gitops/install-gitops.yaml
oc get pods -n openshift-gitops --watch
```
Deploy:
```
oc apply -f gitops/app-of-apps/application-of-apps.yaml
oc patch console.operator.openshift.io cluster --type=json -p '[{"op":"add","path":"/spec/plugins/-","value":"gitops-plugin"}]'
oc get projects --watch | grep hello-world-
```
Add Secrets (optional):
```
oc apply -f manifests/s2i/secrets.yaml -n hello-world-s2i
oc apply -f manifests/deploy/secrets.yaml -n hello-world-deploy
oc get pods -n hello-world-deploy --watch
```
Test
```
oc get route -n hello-world-deploy
```
