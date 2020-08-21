# Update and Apply Kubernetes Configurations

Composite Github action to update k8 configurations with environment variables and then apply them with kubectl.

This requires that kubectl be installed and configured. [Github's hosted runners](https://docs.github.com/en/actions/reference/software-installed-on-github-hosted-runners) already have kubectl installed. If you're using StrongDM to manage access to your k8 cluster, then checkout [configure-kubectl-with-strongdm](https://github.com/marketplace/actions/configure-kubectl-with-strongdm).

# Usage

See [action.yaml](action.yaml)

## Example usage

Given a deployment yaml that is committed to the codebase.

`k8/deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment
  labels:
    app: my-app
  annotations:
    kubernetes.io/change-cause: ${CHANGE_CAUSE}
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: api
        image: xxxxxxxxxxxx.dkr.ecr.us-west-2.amazonaws.com/unicorn/my-app:${IMAGE_TAG}
        ports:
        - containerPort: 80
```

`k8/config-staging.yaml`
```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: my-app
  namespace: default
data:
  APP_ENV: 'staging'
  SOME_CONFIG: 'hello
```

This action uses `envsubst` to replace values in form `$VARIABLE` or `${VARIABLE}` that are found in the list of configuration files. If an ENV variable is not defined for a replacement, then it will be replaced with and empty string. The k8 templates do not need to define any replacements, like the `k8/config-staging.yaml` above. It's still useful to apply these during the deployment so that configuration can be managed through git.

The order that you list the files for `k8-config-file-paths` is the order that they will be applied to your cluster. So, you probably want to apply any configMap updates before applying the deployment.

The files will be applied to kubectl's current-context, so make sure to set it to the correct one before this action runs.

```yaml
- name: Switch to staging cluster
  run: kubectl config use-context staging
- uses: swdotcom/update-and-apply-kubernetes-configs@v1
  with:
    k8-config-file-paths: |
      k8/config-staging.yaml
      k8/deployment.yaml
  env:
    IMAGE_TAG: ${{ github.sha }}
    CHANGE_CAUSE: ${{ github.event.release.tag_name }}
```

Config files can also be defined on one line with a space delimiter.

```yaml
- name: Switch to staging cluster
  run: kubectl config use-context staging
- uses: swdotcom/update-and-apply-kubernetes-configs@v1
  with:
    k8-config-file-paths: k8/config-staging.yaml k8/deployment.yaml
  env:
    IMAGE_TAG: ${{ github.sha }}
    CHANGE_CAUSE: ${{ github.event.release.tag_name }}
```
