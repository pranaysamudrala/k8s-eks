# Postgres

This Documentation is regarding deployment of High Availability Postgres Database using Kustomize in Kubernetes EKS cluster using the GitHub repository mentioned below.

Reference links:-

<https://github.com/CrunchyData/postgres-operator-examples>


Steps for Deployment:


1. ###  **Create a directory and clone the public git repo from the directory:**

```bash
mkdir postgres
cd postgres
```

```bash
git clone https://github.com/CrunchyData/postgres-operator-examples.git
```



2. ### **Navigate to the Specific Directory and Make the Necessary Changes:**

```bash
cd postgres-operator-examples/kustomize/install/rbac/cluster
```

Modify the `service_account.yaml` file by adding the EBS role ARN. This will allow Postgres to create EBS volumes for storing data.

`vi service_account.yaml`

```yaml
apiVersion: v1
 kind: ServiceAccount
 metadata:
   name: pgo
+  annotations:
+    eks.amazonaws.com/role-arn: arn:aws:iam::<aws-account-id>:role/AmazonEKS_EBS_CSI_DriverRole
```

**Explanation:** Replace the ARN value with the EBS role ARN you created while enabling the EBS addon for the Kubernetes cluster.



3. ### **Modify Other Postgres Configurations:**

Navigate to the following directory to modify the `postgres.yaml` file as specified:

```bash
cd ../../../postgres/
```

`vi postgres.yaml`

```yaml
piVersion: postgres-operator.crunchydata.com/v1beta1
kind: PostgresCluster
metadata:
  name: <cluster-name>  # Change name as per your choice
spec:
  postgresVersion: 17
  users:
    - name: postgres  # Changed from 'rhino' to 'postgres'
      databases:
        - testdb  # Changed from 'zoo' to 'testdb'
  instances:
    - name: instance1
      dataVolumeClaimSpec:
        accessModes:
        - "ReadWriteOnce"
        resources:
          requests:
            storage: 20Gi  # Changed storage size to 20Gi
  backups:
    pgbackrest:
      repos:
      - name: repo1
        volume:
          volumeClaimSpec:
            accessModes:
            - "ReadWriteOnce"
            resources:
              requests:
                storage: 20Gi  # Changed storage size to 20Gi
```



4. ### **Create the Namespace for Postgres:**

Use the following command to create a namespace for Postgres:

```bash
cd ../..
kubectl apply -k kustomize/install/namespace
```

This will create a namespace named `postgres-operator`.



5. ### **Install the Postgres Operator:**

Use the below cmd to install the postgres operator:

```bash
kubectl apply --server-side -k kustomize/install/default
```

Expected Output for `kubectl get pods -n postgres-operator`:

```bash
NAME            READY   STATUS    RESTARTS   AGE
pgo-<random>    1/1     Running   0          1m
```



6. ### **Install the Postgres Instance and its Backup Repository:**

Apply the Postgres configurations and create the backup repo host:

```bash
kubectl apply -k kustomize/postgres
```

Expected Output for `kubectl get pods -n postgres-operator`:

NAME                                              READY   STATUS    RESTARTS   AGE

<cluster-name>-instance1-f4h9-0       4/4     Running            0               1m

<cluster-name>-repo-host-0                2/2     Running            0                1m

pgo-84c88fcf85-lwpgq                  1/1     Running             0               4m



7. ### **Retrieve the Password for the** `postgres` **User:**

To access the Postgres database, retrieve the password for the default `postgres` user using the following command:

```javascript
kubectl get secret <cluster-name>-pguser-postgres -n postgres-operator -o jsonpath="{.data.password}" | base64 --decode
```



8. ### **Connect to the Postgres Database:**

After installing psql locally, use the following command to connect to the database via port-forwarding to the <cluster-name>-instance1-\* pod or via LoadBalancer (if applicable):

```javascript
psql -h localhost -U postgres
```



9. ### **Create User:**

By default only user exist in the Postgres database. Create the user with the following command:

```javascript
CREATE USER user WITH PASSWORD "passwordhere';
```



10. ### **Create Database and Grant Database Privileges to the User:**

The user will not have the privilege to create databases by default. To create a database and grant access to user, log in as `postgres` and run:

```javascript
CREATE DATABASE mydatabase OWNER user;
```


```javascript
GRANT ALL PRIVILEGES ON DATABASE mydatabase TO user;
```



11. ### **Configure a Network Load Balancer (NLB) for Postgres (Only on requirement):**

If you want to expose the Postgres service through a Network Load Balancer (NLB), apply the following service configuration:\n`vi postgres-svc.nlb.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: pgo-service
  namespace: postgres-operator
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
spec:
  type: LoadBalancer # Use LoadBalancer or NodePort if external access is required
  selector:
    postgres-operator.crunchydata.com/patroni: <cluster-name>-ha
  ports:
    - protocol: TCP
      port: 5432
      targetPort: 5432    # Port inside the container
```

To create the NLB use below cmd:

```bash
kubectl apply -f postgres-svc-nlb.yaml -n postgres-operator
```

Expected Output for: `kubectl get svc -n postgres-operator`

```bash
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP                                             PORT(S)          AGE
pgo-service     LoadBalancer   172.20.49.105   <external-ip>                                            5432/TCP         5m
```
