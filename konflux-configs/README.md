## Previewing Changes
To preview your changes, use the Kustomize CLI:
```
kustomize build overlay/prod
```

## Applying configuration to cluster
Before applying changes, ensure you have access to the Konflux cluster. Then, execute the following command:
```
oc apply -k overlay/prod
```
