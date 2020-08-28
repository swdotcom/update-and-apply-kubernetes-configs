![Test Action](https://github.com/swdotcom/update-and-apply-kubernetes-configs/workflows/Test%20Action/badge.svg)

# Update and Apply Kubernetes Configurations

Composite Github action to update k8 configurations with environment variables and then apply them with kubectl.

This requires that kubectl be installed and configured. [Github's hosted runners](https://docs.github.com/en/actions/reference/software-installed-on-github-hosted-runners) already have kubectl installed. If you're using StrongDM to manage access to your k8 cluster, then checkout [configure-kubectl-with-strongdm](https://github.com/marketplace/actions/configure-kubectl-with-strongdm).

# Usage

See [action.yaml](action.yaml)

## Example usage

Given kubernetes configurations that are committed to the codebase.

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
  SOME_CONFIG: 'hello'
```

This action uses `envsubst` to replace values in form `$VARIABLE` or `${VARIABLE}` that are found in the list of configuration files. See below for the different replacement options. The k8 templates do not need to define any replacements, like the `k8/config-staging.yaml` above. It's still useful to apply these during the deployment so that configurations can be managed through git.

The order that you list the files for `k8-config-file-paths` is the order that they will be applied to your cluster. So, you probably want to apply any configMap updates before applying the deployment.

**The files will be applied to kubectl's current-context, so make sure to set it to the correct one before this action runs.**

```yaml
- name: Switch to staging cluster
  run: kubectl config use-context staging
- uses: swdotcom/update-and-apply-kubernetes-configs@v1
  with:
    k8-config-file-paths: |
      k8/config-staging.yaml
      k8/deployment.yaml
    replacement-method: all
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
    replacement-method: all
  env:
    IMAGE_TAG: ${{ github.sha }}
    CHANGE_CAUSE: ${{ github.event.release.tag_name }}
```

### Replacement Method - **all (Default)**
Any value matching the format `$VARIABLE` or `${VARIABLE}` will be replaced by a matching ENV variable. If no ENV variable is defined with that name, it will be replaced with nothing. So, it would be best practice to wrap ENV vars in quotes to avoid errors when applying an invalid configuration to kubernetes.

### Replacement Method - **defined**
Any value matching the format `$VARIABLE` or `${VARIABLE}` will be replaced by a matching ENV variable. If no ENV variable is defined with that name, it will be unmodified.

### Replacement Method - **list**
Any value matching the format `$VARIABLE` or `${VARIABLE}` will be replaced by a matching ENV variable. If the ENV variable is not in the predefined list of replacement, it will be unmodified. This is useful for situations where you have a legitimate value that matches `$VAR` or `${VAR}` and you want it to be applied exactly as is to kubernetes.

```yaml
- uses: swdotcom/update-and-apply-kubernetes-configs@v1
  with:
    k8-config-file-paths: k8/config-staging.yaml k8/deployment.yaml
    replacement-method: list
    env-replacement-list: |
      ONLY_ME
  env:
    ONLY_ME: hi
```
