# CHEATSHEET

## Auth

```bash
oc login https://api.<cluster>:6443 --username=myuser
oc login https://api.<cluster>:6443 --token=$(cat token)
oc whoami
oc whoami --show-server
oc whoami -t                          # get current token
oc logout
```

## Projects

```bash
oc projects                           # list all accessible projects
oc project my-app                     # switch to project
oc new-project my-app --display-name="My App"
oc delete project my-app
oc get project my-app -o yaml
```

## Pods

```bash
oc get pods -n my-app
oc get pods -A                        # all namespaces
oc get pods -A | grep -v "Running\|Completed"   # show problem pods
oc describe pod <pod> -n my-app
oc logs <pod> -n my-app
oc logs <pod> -n my-app --previous    # crashed container
oc logs -f <pod> -n my-app            # follow
oc rsh <pod> -n my-app                # remote shell
oc exec <pod> -n my-app -- <command>
oc debug pod/<pod> -n my-app          # debug copy
oc delete pod <pod> -n my-app
oc adm top pods -n my-app             # resource usage
```

## Deployments

```bash
oc get deployments -n my-app
oc scale deployment my-app --replicas=3 -n my-app
oc rollout status deployment/my-app -n my-app
oc rollout undo deployment/my-app -n my-app
oc rollout history deployment/my-app -n my-app
oc set image deployment/my-app my-app=registry/image:tag -n my-app
oc set env deployment/my-app ENV_VAR=value -n my-app
```

## Routes & Services

```bash
oc get routes -n my-app
oc get svc -n my-app
oc expose svc/my-app -n my-app
oc expose svc/my-app --hostname=api.example.com -n my-app
oc get route my-app -n my-app -o jsonpath='{.spec.host}'
oc delete route my-app -n my-app
oc get endpoints -n my-app
```

## Builds

```bash
oc get builds -n my-app
oc start-build my-app -n my-app
oc logs -f bc/my-app -n my-app
oc cancel-build <build-name> -n my-app
```

## Nodes

```bash
oc get nodes
oc describe node <node>
oc adm top nodes
oc adm cordon <node>
oc adm uncordon <node>
oc adm drain <node> --ignore-daemonsets --delete-emptydir-data
oc debug node/<node>                  # shell into node
```

## RBAC

```bash
oc adm policy add-role-to-user admin <user> -n my-app
oc adm policy add-role-to-user view <user> -n my-app
oc adm policy add-role-to-group edit <group> -n my-app
oc adm policy add-cluster-role-to-user cluster-admin <user>
oc adm policy remove-role-from-user edit <user> -n my-app
oc get rolebindings -n my-app
oc get clusterrolebindings | grep <user>
```

## SCCs

```bash
oc get scc
oc adm policy add-scc-to-user anyuid -z <sa> -n my-app
oc adm policy remove-scc-from-user anyuid -z <sa> -n my-app
oc get pod <pod> -o jsonpath='{.metadata.annotations.openshift\.io/scc}' -n my-app
oc adm policy scc-review -f pod.yaml
```

## Cluster

```bash
oc get co                             # cluster operators
oc get clusterversion                 # version + upgrade status
oc adm upgrade                        # available updates
oc get mcp                            # machine config pools
oc get nodes -o wide                  # nodes with IPs
oc cluster-info
oc adm must-gather --dest-dir=./diag  # collect diagnostics
```

## Secrets & ConfigMaps

```bash
oc create secret generic my-secret --from-literal=key=value -n my-app
oc create secret docker-registry pull-secret \
  --docker-server=reg --docker-username=u --docker-password=p -n my-app
oc create configmap my-cm --from-literal=key=value -n my-app
oc create configmap my-cm --from-file=config.yaml -n my-app
oc get secret my-secret -n my-app -o jsonpath='{.data.key}' | base64 -d
oc set env deployment/my-app --from=secret/my-secret -n my-app
```

## Useful one-liners

```bash
# All non-running pods
oc get pods -A --field-selector=status.phase!=Running | grep -v Completed

# Events sorted by time
oc get events -n my-app --sort-by='.lastTimestamp' | tail -20

# Delete all completed pods
oc delete pods --field-selector=status.phase==Succeeded -A

# Get all images running in a namespace
oc get pods -n my-app -o jsonpath='{.items[*].spec.containers[*].image}' | tr ' ' '\n' | sort -u

# Current token
oc whoami -t

# Get API server URL
oc whoami --show-server
```
