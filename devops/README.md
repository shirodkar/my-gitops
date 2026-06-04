Install GitOps:
```
oc apply -f gitops/install-gitops.yaml
oc get pods -n openshift-gitops --watch
```
Setup Infra:
```
oc apply -f gitops/infra/application-infra.yaml
oc patch console.operator.openshift.io cluster --type=json -p '[{"op":"add","path":"/spec/plugins/-","value":"gitops-plugin"}]'
oc get projects --watch | grep hello-world-
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
