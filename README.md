# MongoDB Kanister Hooks for KubeVirt VMs on OpenShift

This document covers the setup of Kasten K10 pre/post backup hooks for a MongoDB instance running inside a KubeVirt VM on OpenShift. The hooks use a Kanister Blueprint to issue `fsyncLock` and `fsyncUnlock` commands around the Kasten snapshot, ensuring crash-consistent backups.

---

## Architecture

```
Kasten K10 (kasten-io)
    │
    ├── backupPrehook  ──► KubeTask Pod ──► mongo-kanister ClusterIP SVC ──► virt-launcher pod ──► RHEL VM ──► mongod
    │
    └── backupPosthook ──► KubeTask Pod ──► mongo-kanister ClusterIP SVC ──► virt-launcher pod ──► RHEL VM ──► mongod
```

Rather than exec-ing into the VM (which is not possible with KubeVirt), the blueprint spawns a throwaway `kanisterio/mongodb` pod that connects to MongoDB over the network via a ClusterIP service. The service selects the virt-launcher pod using the `vm.kubevirt.io/name` label.

---

## Prerequisites

- OpenShift with KubeVirt/OpenShift Virtualization installed
- Kasten K10 installed in the `kasten-io` namespace
- A RHEL VM running MongoDB, exposed via a ClusterIP service
- MongoDB configured with authentication enabled and listening on `0.0.0.0:27017`

---

## Components

| Component | Name | Namespace |
|---|---|---|
| Kanister Blueprint | `mongo-vm-hooks` | `kasten-io` |
| ClusterIP Service | `mongo-kanister` | `ocpv-rhel9` |
| Credentials Secret | `mongo-kanister` | `ocpv-rhel9` |
| VirtualMachine | `rhel-10` | `ocpv-rhel9` |

---

## MongoDB Installation (RHEL 10)

### 1. Add the MongoDB repository

```bash
cat > /etc/yum.repos.d/mongodb-org-8.0.repo << 'EOF'
[mongodb-org-8.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/9/mongodb-org/8.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://pgp.mongodb.com/server-8.0.asc
EOF
```

> Note: RHEL 10 is not yet in MongoDB's official repo matrix. The RHEL 9 baseurl is compatible.

### 2. Install and start MongoDB

```bash
dnf install -y mongodb-org
systemctl enable --now mongod
```

### 3. Configure mongod.conf

Edit `/etc/mongod.conf` to bind on all interfaces and enable auth:

```yaml
net:
  port: 27017
  bindIp: 0.0.0.0

security:
  authorization: enabled
```

### 4. Create the root user

With auth enabled but no users yet, MongoDB allows a one-time unauthenticated connection from localhost (the localhost exception):

```bash
mongosh admin --eval "
db.createUser({
  user: 'root',
  pwd: 'YOUR_PASSWORD',
  roles: [ { role: 'root', db: 'admin' } ]
})
"
```

### 5. Restart and verify

```bash
systemctl restart mongod

mongosh --authenticationDatabase admin -u root -p YOUR_PASSWORD \
  --eval "db.runCommand({ connectionStatus: 1 })"
```

Expected output includes `"authenticated" : true`.

---

## Kubernetes Resources

### Secret

Stores the MongoDB root password. Must be in the same namespace as the VM.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongo-kanister
  namespace: ocpv-rhel9
type: Opaque
stringData:
  mongodb-root-password: "YOUR_PASSWORD"
```

> **Warning**: Do not commit this file to source control with a plaintext password. Use Sealed Secrets, Vault, or equivalent.

### ClusterIP Service

Exposes the MongoDB port from the virt-launcher pod. The selector uses `vm.kubevirt.io/name` which KubeVirt automatically applies to the virt-launcher pod.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongo-kanister
  namespace: ocpv-rhel9
spec:
  selector:
    vm.kubevirt.io/name: rhel-10
  ports:
  - name: mongodb
    protocol: TCP
    port: 27017
    targetPort: 27017
  type: ClusterIP
```

Verify the endpoint is populated after applying:

```bash
oc get endpoints mongo-kanister -n ocpv-rhel9
```

You should see a pod IP listed under `ENDPOINTS`. If it shows `<none>`, the selector is not matching — check the virt-launcher pod labels:

```bash
oc get pods -n ocpv-rhel9 --show-labels | grep virt-launcher
```

### Kanister Blueprint

The blueprint runs in `kasten-io` and spawns a KubeTask pod in the VM namespace for each hook phase.

```yaml
actions:
  backupPrehook:
    kind: ""
    name: ""
    phases:
    - args:
        command:
        - bash
        - -o
        - errexit
        - -o
        - pipefail
        - -c
        - |
          export MONGODB_ROOT_PASSWORD='{{ index .Phases.lockMongo.Secrets.mongoDbSecret.Data "mongodb-root-password" | toString }}'
          mongosh --authenticationDatabase admin \
            -u root \
            -p "${MONGODB_ROOT_PASSWORD}" \
            --host mongo-kanister.{{ .Object.metadata.namespace }}.svc.cluster.local \
            --eval "db.fsyncLock()"
        image: ghcr.io/kanisterio/mongodb:0.116.0
        namespace: '{{ .Object.metadata.namespace }}'
      func: KubeTask
      name: lockMongo
      objects:
        mongoDbSecret:
          apiVersion: ""
          group: ""
          kind: Secret
          name: mongo-kanister
          namespace: '{{ .Object.metadata.namespace }}'
          resource: ""
  backupPosthook:
    kind: ""
    name: ""
    phases:
    - args:
        command:
        - bash
        - -o
        - errexit
        - -o
        - pipefail
        - -c
        - |
          export MONGODB_ROOT_PASSWORD='{{ index .Phases.unlockMongo.Secrets.mongoDbSecret.Data "mongodb-root-password" | toString }}'
          mongosh --authenticationDatabase admin \
            -u root \
            -p "${MONGODB_ROOT_PASSWORD}" \
            --host mongo-kanister.{{ .Object.metadata.namespace }}.svc.cluster.local \
            --eval "db.fsyncUnlock()"
        image: ghcr.io/kanisterio/mongodb:0.116.0
        namespace: '{{ .Object.metadata.namespace }}'
      func: KubeTask
      name: unlockMongo
      objects:
        mongoDbSecret:
          apiVersion: ""
          group: ""
          kind: Secret
          name: mongo-kanister
          namespace: '{{ .Object.metadata.namespace }}'
          resource: ""
apiVersion: cr.kanister.io/v1alpha1
kind: Blueprint
metadata:
  name: mongo-vm-hooks
  namespace: kasten-io
```

---

## VM Annotation

Annotate the VirtualMachine object so Kasten picks up the blueprint automatically:

```bash
oc annotate virtualmachine rhel-10 -n ocpv-rhel9 \
  kanister.kasten.io/blueprint='mongo-vm-hooks'
```

Or add it directly to the VM manifest:

```yaml
metadata:
  name: rhel-10
  namespace: ocpv-rhel9
  annotations:
    kanister.kasten.io/blueprint: mongo-vm-hooks
```

---

## Connectivity Test

Before running the policy, verify the kanister job pod can reach MongoDB across the service:

```bash
oc run mongo-test -n ocpv-rhel9 --rm -it \
  --image=ghcr.io/kanisterio/mongodb:0.116.0 -- \
  mongosh --host mongo-kanister.ocpv-rhel9.svc.cluster.local \
  --authenticationDatabase admin -u root -p YOUR_PASSWORD \
  --eval "db.runCommand({ connectionStatus: 1 })"
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `"labels" not found` template error | Blueprint referencing a label that doesn't exist on the VM object | Hardcode the secret name instead of using label lookup |
| `ECONNREFUSED` on ClusterIP | Service selector not matching virt-launcher pod | Check `oc get endpoints` and verify `vm.kubevirt.io/name` label |
| `ENDPOINTS: <none>` | Wrong selector label (e.g. `kubevirt.io/domain` vs `vm.kubevirt.io/name`) | Use `oc get pods --show-labels` to confirm correct label key |
| `Authentication failed` | User not created, or created before auth enabled | Use localhost exception to recreate user after auth is enabled |
| `mongod` only on `127.0.0.1` | Default bindIp not changed | Set `bindIp: 0.0.0.0` in `/etc/mongod.conf` and restart |
