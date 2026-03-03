# k8s ConfigMaps

## Goal

Learn how `k8s ConfigMaps` work and test them out.

## Prerequisites

A `k8s` cluster.

## Introduction

We want to use `k8s ConfigMaps` to provide configuration to our applications. `ConfigMaps` must not contain sensitive data. Use `k8s secrets` for that purpose.

## Creation

Let's create a complex enough `ConfigMap` to find out what can we put in it:
```yaml
# config-map.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-config
data:
  # property-like keys; each key maps to a simple value
  version: "3.1.4"
  main_app_name: "myapp"

  # file-like keys
  startup.script: |
    #!/bin/sh
    
    echo abc
    echo 111
```
Let's first understand what we have here. Configurations are key-value pairs. The first two of them have simple values, while the 3rd one has a multi-line value, which is basically a script in my case. The multi-line values are seen as a blob, so `k8s` itself does not do any parsing of the content.

Now we can apply the `ConfigMap`:
```sh
kubectl apply -f config-map.yaml
```

## Usage

Let's create a deployment to use the `ConfigMap`:
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-config-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-config-app
  template:
    metadata:
      labels:
        app: test-config-app
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ["/bin/sh", "-c"]
        args:
          - |
            echo "VERSION=$VERSION"
            echo "MAIN_APP_NAME=$MAIN_APP_NAME"
            echo ""
            echo "Executing startup.script:"
            chmod +x /etc/config/startup.script
            /etc/config/startup.script
            sleep 3600
        env:
        - name: VERSION
          valueFrom:
            configMapKeyRef:
              name: test-config
              key: version
        - name: MAIN_APP_NAME
          valueFrom:
            configMapKeyRef:
              name: test-config
              key: main_app_name
        volumeMounts:
        - name: test-config-volume
          mountPath: /etc/config
          readOnly: true
      volumes:
      - name: test-config-volume
        configMap:
          name: test-config
```
Since the first 2 configurations have simple values, we usually use them as environment variables in the container. The 3rd configuration, which has the content of a script, is usually mounted as a file. Of course, we are free to make such choices as we want and as it fits better for any specific scenario. 
Now we can and apply the deployment:
```sh
kubectl apply -f deployment.yaml
```
Now let's get the logs by calling:
```sh
kubectl logs deployment/test-config-deployment
```
This returns:
```sh
VERSION=3.1.4
MAIN_APP_NAME=myapp

Executing startup.script:
chmod: /etc/config/startup.script: Read-only file system
/bin/sh: /etc/config/startup.script: Permission denied
```
We notice that the environment variables are printed out correctly. Probably the unexpected part is the execution of the script, which failed with *Permission denied*. `ConfigMap` volumes are mounted read-only, and files are mounted with mode 0644 by default (not executable).. As a result, *chmod* fails, and direct execution fails due to missing execute permissions. Even if you omit *readOnly: true* from the *volumeMounts*, `k8s` still mounts `ConfigMap` volumes read-only; setting it just makes the intent explicit. A better strategy is to make a copy of the mounted file, and in our case of a script, give execution rights to the copy and run it, so let's update our deployment:
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-config-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-config-app
  template:
    metadata:
      labels:
        app: test-config-app
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ["/bin/sh", "-c"]
        args:
          - |
            echo "VERSION=$VERSION"
            echo "MAIN_APP_NAME=$MAIN_APP_NAME"
            echo ""
            echo "Create /work dir, copy script from read-only mount to /work and execute:"
            mkdir /work
            cp /etc/config/startup.script /work/startup.sh
            chmod +x /work/startup.sh
            /work/startup.sh
            sleep 3600
        env:
        - name: VERSION
          valueFrom:
            configMapKeyRef:
              name: test-config
              key: version
        - name: MAIN_APP_NAME
          valueFrom:
            configMapKeyRef:
              name: test-config
              key: main_app_name
        volumeMounts:
        - name: test-config-volume
          mountPath: /etc/config
          readOnly: true
      volumes:
      - name: test-config-volume
        configMap:
          name: test-config
```
, and apply it again:
```sh
kubectl apply -f deployment.yaml
```
If we now query the logs:
```sh
kubectl logs deployment/test-config-deployment
```
, we get:
```sh
VERSION=3.1.4
MAIN_APP_NAME=myapp

Create /work dir, copy script from read-only mount to /work and execute:
abc
111
```
Now we notice that our script ran successfully.

Basically this is how you can use `ConfigMaps`. Since we have a scenario that can get a bit messy let's explore it further.

## Advanced

If we care about security, our main container usually has a configuration like:
```yaml
        securityContext:
          readOnlyRootFilesystem: true
```
, so our deployment would look like:
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-config-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-config-app
  template:
    metadata:
      labels:
        app: test-config-app
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ["/bin/sh", "-c"]
        args:
          - |
            echo "VERSION=$VERSION"
            echo "MAIN_APP_NAME=$MAIN_APP_NAME"
            echo ""
            echo "Create /work dir, copy script from read-only mount to /work and execute:"
            mkdir /work
            cp /etc/config/startup.script /work/startup.sh
            chmod +x /work/startup.sh
            /work/startup.sh
            sleep 3600
        securityContext:
          readOnlyRootFilesystem: true
        env:
        - name: VERSION
          valueFrom:
            configMapKeyRef:
              name: test-config
              key: version
        - name: MAIN_APP_NAME
          valueFrom:
            configMapKeyRef:
              name: test-config
              key: main_app_name
        volumeMounts:
        - name: test-config-volume
          mountPath: /etc/config
          readOnly: true
      volumes:
      - name: test-config-volume
        configMap:
          name: test-config
```
If we now apply it again:
```sh
kubectl apply -f deployment.yaml
```
, and query the logs:
```sh
kubectl logs deployment/test-config-deployment
```
, we get:
```sh
VERSION=3.1.4
MAIN_APP_NAME=myapp

Create /work dir, copy script from read-only mount to /work and execute:
mkdir: can't create directory '/work': Read-only file system
cp: can't create '/work/startup.sh': No such file or directory
chmod: /work/startup.sh: No such file or directory
/bin/sh: /work/startup.sh: not found
```
So we get some errors because we are not allowed to create new directories under root.
We can fix this by mounting a second empty volume under */work*, and obtain the */work* path that way. Let's update our deployment:
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-config-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-config-app
  template:
    metadata:
      labels:
        app: test-config-app
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ["/bin/sh", "-c"]
        args:
          - |
            echo "VERSION=$VERSION"
            echo "MAIN_APP_NAME=$MAIN_APP_NAME"
            echo ""
            echo "Copy script from read-only mount to /work and execute:"
            cp /etc/config/startup.script /work/startup.sh
            chmod +x /work/startup.sh
            /work/startup.sh
            sleep 3600
        securityContext:
          readOnlyRootFilesystem: true
        env:
        - name: VERSION
          valueFrom:
            configMapKeyRef:
              name: test-config
              key: version
        - name: MAIN_APP_NAME
          valueFrom:
            configMapKeyRef:
              name: test-config
              key: main_app_name
        volumeMounts:
        - name: test-config-volume
          mountPath: /etc/config
          readOnly: true
        - name: workdir
          mountPath: /work
      volumes:
      - name: test-config-volume
        configMap:
          name: test-config
      - name: workdir
        emptyDir: {}
```
, and apply it again:
```sh
kubectl apply -f deployment.yaml
```
If we now query the logs:
```sh
kubectl logs deployment/test-config-deployment
```
, we get:
```sh
VERSION=3.1.4
MAIN_APP_NAME=myapp

Copy script from read-only mount to /work and execute:
abc
111
```
, so the script successfully runs again.

From an architectural perspective it is better to move the preparation of the script into an init container. Details about this are another story for another day. For now let's change our deployment to handle the creation of */work* in an init container:
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-config-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-config-app
  template:
    metadata:
      labels:
        app: test-config-app
    spec:
      initContainers:
      - name: copy-startup-script
        image: busybox
        command: ["/bin/sh", "-c"]
        args:
          - |
            cp /etc/config/startup.script /work/startup.sh
            chmod +x /work/startup.sh
        volumeMounts:
        - name: test-config-volume
          mountPath: /etc/config
          readOnly: true
        - name: workdir
          mountPath: /work
      containers:
      - name: busybox
        image: busybox
        command: ["/bin/sh", "-c"]
        args:
          - |
            echo "VERSION=$VERSION"
            echo "MAIN_APP_NAME=$MAIN_APP_NAME"
            echo ""
            echo "Execute script:"
            /work/startup.sh
            sleep 3600
        securityContext:
          readOnlyRootFilesystem: true
        env:
        - name: VERSION
          valueFrom:
            configMapKeyRef:
              name: test-config
              key: version
        - name: MAIN_APP_NAME
          valueFrom:
            configMapKeyRef:
              name: test-config
              key: main_app_name
        volumeMounts:
        - name: test-config-volume
          mountPath: /etc/config
          readOnly: true
        - name: workdir
          mountPath: /work
      volumes:
      - name: test-config-volume
        configMap:
          name: test-config
      - name: workdir
        emptyDir: {}
```
, and apply it again:
```sh
kubectl apply -f deployment.yaml
```
If we now query the logs:
```sh
kubectl logs deployment/test-config-deployment -c busybox
```
You can point to a specific container with *-c*, but by default `kubectl` shows the main container logs and tells you which one it chose.
The output is:
```sh
VERSION=3.1.4
MAIN_APP_NAME=myapp

Execute script:
abc
111
```
, so the script successfully runs. Note that we used an init container, where the */work* directory was created automatically by the volume mount, and where we copied the script to the desired location and gave it execution rights. In the main container we can now just run the script after we mount the same volume, so that it is synced between the init container and the main container.

`ConfigMaps` update works a bit different, depending on usage:
- Volume mounts: If you update a `ConfigMap`, the mounted files in running pods are automatically updated (though it may take up to ~60 seconds due to the kubelet's periodic sync cache).
- Environment variables: These are not updated right away. If you update a `ConfigMap`, any running pods will continue using the old value until the pod is restarted.
This is an important distinction when deciding between volume mounts and environment variables for your use case.

## Cleanup

If you followed along, run the command to clean everything up:
```sh
kubectl delete -f deployment.yaml
kubectl delete -f config-map.yaml
```
