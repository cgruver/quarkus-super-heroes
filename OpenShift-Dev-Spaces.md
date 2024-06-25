# Run Quarkus Super Heroes in OpenShift Dev Spaces with Podman Compose

## Allow privileged rootless pods in Dev Spaces

### Create the following SCC as a cluster-admin

```bash
cat << EOF | oc apply -f -
apiVersion: security.openshift.io/v1
metadata:
  name: nested-podman-scc
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegeEscalation: true
allowPrivilegedContainer: true
allowedCapabilities:
- SETUID
- SETGID
defaultAddCapabilities: null
fsGroup:
  type: MustRunAs
groups: []
kind: SecurityContextConstraints
priority: null
readOnlyRootFilesystem: false
requiredDropCapabilities:
- KILL
- MKNOD
runAsUser:
  type: MustRunAsRange
seLinuxContext:
  type: MustRunAs
supplementalGroups:
  type: RunAsAny
users: []
volumes:
- configMap
- downwardAPI
- emptyDir
- persistentVolumeClaim
- projected
- secret
EOF
```

### Modify the CheCluster CR to use the new SCC

__Note:__ If you already have a `CheCluster` CR created, then you can just modify it.

Change the entry for `containerBuildConfiguration` to look like:

```bash
containerBuildConfiguration:
  openShiftSecurityContextConstraint: nested-podman-scc
```

__Example of a new install:__

```bash
cat << EOF | oc apply -f -
apiVersion: v1                      
kind: Namespace                 
metadata:
  name: devspaces
---           
apiVersion: org.eclipse.che/v2 
kind: CheCluster   
metadata:              
  name: devspaces  
  namespace: devspaces
spec:                         
  components:                  
    cheServer:      
      debug: false
      logLevel: INFO
    metrics:                
      enable: true
    pluginRegistry:
      openVSXURL: https://open-vsx.org
  containerRegistry: {}      
  devEnvironments:       
    startTimeoutSeconds: 300
    secondsOfRunBeforeIdling: -1
    maxNumberOfWorkspacesPerUser: -1
    maxNumberOfRunningWorkspacesPerUser: 5
    containerBuildConfiguration:
      openShiftSecurityContextConstraint: nested-podman-scc
    disableContainerBuildCapabilities: false
    defaultComponents:
    - name: dev-tools
      container:
        image: quay.io/cgruver0/che/dev-tools:latest
        memoryLimit: 6Gi
        mountSources: true
    defaultEditor: che-incubator/che-code/latest
    defaultNamespace:
      autoProvision: true
      template: <username>-devspaces
    secondsOfInactivityBeforeIdling: 1800
    storage:
      pvcStrategy: per-workspace
  gitServices: {}
  networking: {}  
EOF
```

## Create a Workspace

1. Log into OpenShift Dev Spaces as a non-cluster-admin user.

1. Create a new workspace from the git URL of this repository

## Run the demo app

1. Open a new terminal in the workspace

```
export API_BASE_URL=https://$(oc get route ${DEVWORKSPACE_ID}-dev-tools-8082-https-fights -o jsonpath={.spec.host})
podman compose -f /projects/quarkus-super-heroes/deploy/docker-compose/dev-spaces-java17.yml -f /projects/quarkus-super-heroes/deploy/docker-compose/dev-spaces-monitoring.yml up --remove-orphans
```
