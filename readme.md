## EGRESS policies for Kubernetes and ProjectCalico
In kubernetes network policies are namespaced (so you have to apply them to every namespace). So let's create a namespace to investigate things.
```
kubectl create ns my-demo-app

kubectl run -it --rm --image xxradar/hackon -l allow-internet-egress=true -n my-demo-app my-app -- bash
```

### 1. Default Deny Egress policy

```
cat >default-deny-egress.yaml <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all-egress
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    ports:
    - protocol: TCP
      port: 53
    - protocol: UDP
      port: 53
EOF
```
```
kubectl apply -n my-demo-app -f default-deny-egress.yaml 
```

### 2. Allow access to the internet on port 80 and 443 for selected pods
```
cat >allow-80-443-egress.yaml <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-80-443-egress
spec:
  podSelector:
    matchLabels:
      allow-internet-egress: "true"
  policyTypes:
  egress:
  - to:
    ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 443
EOF
```
```
kubectl apply -n my-demo-app -f allow-80-443-egress.yaml 
```
Please note that network policies are applied even on running pods labeled with **allow-internet-egress="true"**

### 3. Deny Egress access via Calico Network sets

```
cat >networkset-external-api.yaml <<EOF
apiVersion: projectcalico.org/v3
kind: NetworkSet
metadata:
  name: networkset-api-access
  labels:
    role: api
spec:
  nets:
  - 198.199.124.250/24
  - 95.85.54.44/32
EOF
```
```
calicoctl apply -n my-demo-app -f networkset-external-api.yaml
```
```
cat >deny-external-api.yaml <<EOF
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: external-api-access
spec:
  order: 100
  selector:
    allow-internet-egress == "true"
  types:
    - Egress
  egress:    
    - action: Deny
      destination:
        selector: role == "api"
EOF
```
```
calicoctl apply -n my-demo-app -f deny-external-api.yaml
```

### 4. Deny Egress access via Calico Global Network sets
```
cat >global-networkset-external-api.yaml <<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkSet
metadata:
  name: global-networkset-external-api
  labels:
    role: api-global
spec:
  nets:
  - 198.199.124.250/24
  - 95.85.54.44/32
EOF
```
```
calicoctl apply -f global-networkset-external-api.yaml
```
```
cat >global-deny-external-api.yaml <<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: global-external-api-access
spec:
  order: 80
  selector:
    allow-internet-egress == "true"
  types:
    - Egress
  egress:    
    - action: Deny
      destination:
        selector: role == "api-global"
EOF
```
```
calicoctl apply -f global-deny-external-api.yaml
```
We can test this spinning up a pod in another namespace (ex. default)
```
kubectl run -it --rm --image xxradar/hackon -l allow-internet-egress=true my-app -- bash
```
### 5. Allow Egress access based on FQDN
In order to continue, let's delete the previous policies.
```
kubectl delete networkpolicies  allow-80-443-egress -n my-demo-app 
calicoctl delete networkpolicies  external-api-access -n my-demo-app 
calicoctl delete globalnetworkpolicies global-external-api-access
```
Let's create a **namespaced** network policy allowing only a specific fqdn
```
cat >dns-deny-external-api.yaml <<EOF
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: dns-deny-external-api
spec:
  selector: allow-internet-egress == "true"
  egress:
    - action: Allow
      source: {}
      destination:
        domains:
          - "*.radarhack.com"
  types:
    - Egress
EOF
```
```
kubectl apply  -n my-demo-app -f dns-deny-external-api.yaml
```
