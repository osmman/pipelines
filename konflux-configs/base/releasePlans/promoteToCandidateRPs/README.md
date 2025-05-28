# Promote to Candidate Release Plans

This directory contains the release plans used in the automated build process for RHTAS. To simplify management, these release plans are configured using Kustomize.
There are three release types:

1. Component: Refers to our component applications.
2. Operator: Refers to our operator applications.
3. FBC: Refers to our FBC applications.


```
├── components
│   ├── base
│   └── overlays
├── fbc
│   ├── base
│   └── overlays
├── operator
│   ├── base
│   └── overlays
└── README.md
```

## Configuration Guidelines
Top-Level Configurations:
    Changes should be made in the <type>/base/releasePlan directory.

Individual Changes:
    Apply modifications in the <type>/overlays/ directory.

Always commit any changes to ensure that the current state of the release plans on the Konflux cluster is accurately reflected in these.

## Previewing Changes
To preview your changes, use the Kustomize CLI:
```
kustomize build <type>/overlays
```

## Applying Release Plans
Before applying changes, ensure you have access to the Konflux cluster. Then, execute the following command:
```
oc apply -k <type>/overlays
```
For updating singular release plans, run:
```
oc apply -k <type>/overlays/<app>
```