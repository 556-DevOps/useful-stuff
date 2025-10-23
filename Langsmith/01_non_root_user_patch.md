# Langsmith helm release : apply patch to run containers as non-root user

## Get users of all pods in langchain namespace
```sh
for pod in $(kubectl get pods -n langchain -o name); do
  pod_name=$(echo "$pod" | cut -d'/' -f2)
  containers=$(kubectl get "$pod" -o jsonpath='{.spec.containers[*].name}')
  
  for container in $containers; do
    uid_gid=$(
      kubectl exec "$pod" -c "$container" -- id 2>/dev/null \
      | grep -oP 'uid=\K[^ ]+|gid=\K[^ ]+' \
      | head -2 \
      | tr '\n' ' '
    )
    echo "$pod_name | $container | $uid_gid"
  done
done
```

Output is like this:

| Pod Name                                           | Container Name       | UID (User)       | GID (Group)      |
|----------------------------------------------------|----------------------|------------------|------------------|
| langsmith-ace-backend-68698f9c5d-hn54r             | ace-backend          | 0 (root)         | 0 (root)         |
| langsmith-backend-7777d57c75-rr72z                 | backend              | 0 (root)         | 0 (root)         |
| langsmith-backend-7777d57c75-wpkj5                 | backend              | 0 (root)         | 0 (root)         |
| langsmith-backend-auth-bootstrap-k8vcr             | auth-bootstrap       | —                | —                |
| langsmith-backend-ch-migrations-pvgqt              | ch-migrations        | —                | —                |
| langsmith-backend-config-migrations-bwj4c           | feedback-migrations  | —                | —                |
| langsmith-backend-fb-migrations-rzhz8              | config-migrations    | —                | —                |
| langsmith-backend-migrations-xprsz                 | pg-migrations        | —                | —                |
| langsmith-clickhouse-0                             | clickhouse           | 0 (root)         | 0 (root)         |
| langsmith-frontend-6c8fcdbf5b-hkmw7                | frontend             | 65532 (nonroot)  | 65532 (nonroot)  |
| langsmith-platform-backend-5f7698fbf7-47f9b        | platform-backend     | 0 (root)         | 0 (root)         |
| langsmith-platform-backend-5f7698fbf7-8g9fp        | platform-backend     | 0 (root)         | 0 (root)         |
| langsmith-platform-backend-5f7698fbf7-j89gc        | platform-backend     | 0 (root)         | 0 (root)         |
| langsmith-playground-6d45fb49d9-59xhm              | playground           | 0 (root)         | 0 (root)         |
| langsmith-queue-54cdf8f4b8-hbczh                   | queue                | 0 (root)         | 0 (root)         |
| langsmith-queue-54cdf8f4b8-lfbfz                   | queue                | 0 (root)         | 0 (root)         |
| langsmith-queue-54cdf8f4b8-xtjbd                   | queue                | 0 (root)         | 0 (root)         |

## Get the statefulsets in langchain namespace
```sh
k get statefulset -n langchain

NAME                   READY   AGE
langsmith-clickhouse   1/1     12m
```

Ensure that only the clickhouse statefulset exists.


## Dry run the upgrade to see what will change
```sh
helm plugin install https://github.com/databus23/helm-diff --version v3.9.8

helm diff upgrade langsmith langchain/langsmith \
  -n langchain \
  --reuse-values \
  -f langsmith-values-non-root-patch.yaml > diff-upgrade.txt
```

Your upgrade is safe for existing persistent stores if you only modify securityContext, podSecurityContext, volumes, volumeMounts, and don’t change the PVC definitions.

ClickHouse Pods may restart, but data should remain intact.

```yaml
-             {}
+             allowPrivilegeEscalation: false
+             capabilities:
+               drop:
+               - ALL
+             readOnlyRootFilesystem: true
+             seccompProfile:
+               type: RuntimeDefault
```

volumeClaimTemplates shouldn't be altered by any means! Quick Search the diff:
```sh
grep -n -A20 -B10 "volumeClaimTemplates" diff-upgrade.txt
```



## Perform the upgrade
```sh
helm upgrade langsmith langchain/langsmith \
  -n langchain \
  --reuse-values \
  -f langsmith-values-non-root-patch.yaml \
  --wait --debug
```

## Get users of all pods in langchain namespace
```sh
for pod in $(kubectl get pods -n langchain -o name); do
  pod_name=$(echo "$pod" | cut -d'/' -f2)
  containers=$(kubectl get "$pod" -o jsonpath='{.spec.containers[*].name}')
  
  for container in $containers; do
    uid_gid=$(
      kubectl exec "$pod" -c "$container" -- id 2>/dev/null \
      | grep -oP 'uid=\K[^ ]+|gid=\K[^ ]+' \
      | head -2 \
      | tr '\n' ' '
    )
    echo "$pod_name | $container | $uid_gid"
  done
done
```


| Pod Name                                           | Container Name       | UID (User)       | GID (Group)      |
|----------------------------------------------------|----------------------|------------------|------------------|
| langsmith-ace-backend-84b6f855d7-xz8d8               | ace-backend         | 1000     | 1000   |
| langsmith-backend-auth-bootstrap-9fg5f               | auth-bootstrap      |          |        |
| langsmith-backend-b5df6bd48-b8csh                    | backend             | 1000     | 1000   |
| langsmith-backend-b5df6bd48-zp2vz                    | backend             | 1000     | 1000   |
| langsmith-backend-ch-migrations-85khv                | ch-migrations       |          |        |
| langsmith-backend-config-migrations-nglf9            | feedback-migrations |          |        |
| langsmith-backend-fb-migrations-5d2jm                | config-migrations   |          |        |
| langsmith-backend-migrations-vq9v8                   | pg-migrations       |          |        |
| langsmith-clickhouse-0                               | clickhouse          | 101(clickhouse) | 101(clickhouse) |
| langsmith-frontend-5476c7dd7b-fp8zr                  | frontend            | 1000     | 1000   |
| langsmith-platform-backend-fd7c99bc4-8znl5           | platform-backend    | 1000     | 1000   |
| langsmith-platform-backend-fd7c99bc4-b6xn4           | platform-backend    | 1000     | 1000   |
| langsmith-platform-backend-fd7c99bc4-q7rc9           | platform-backend    | 1000     | 1000   |
| langsmith-playground-7ff8bb754b-lk5c2                | playground          | 1000     | 1000   |
| langsmith-queue-7b75f565df-4whpf                     | queue               | 1000     | 1000   |
| langsmith-queue-7b75f565df-ftrmb                     | queue               | 1000     | 1000   |
| langsmith-queue-7b75f565df-hh9cv                     | queue               | 1000     | 1000   |


