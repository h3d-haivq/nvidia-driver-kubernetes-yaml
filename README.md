# YAML file for NVIDIA Device support for Kubernetes

Manage and schedule your nodes with ease. No more hard-config by `NVIDIA_VISIBLE_DEVICES=<ids>`

## Prequesites:
   - Have `docker` installed.
   - Have `nvidia-docker2` installed (this method is deprecated by docker, but kubernetes is still using it then it's fine) 
   ```bash
   curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | \
   apt-key add -
   distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
   curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | \
      tee /etc/apt/sources.list.d/nvidia-docker.list
   apt-get update && apt-get install nvidia-docker2 -y
   ```
   - Disabled some modules which will crash NVIDIA Driver (skip this step if you are not planning to use dockerized driver)
   ```bash
   sed -i 's/^#root/root/' /etc/nvidia-container-runtime/config.toml
   tee /etc/modules-load.d/ipmi.conf <<< "ipmi_msghandler"
   tee /etc/modprobe.d/blacklist-nouveau.conf <<< "blacklist nouveau"
   tee -a /etc/modprobe.d/blacklist-nouveau.conf <<< "options nouveau modeset=0"
   ```
   - Have the docker runtime set to `nvidia` instead of `runc`
   Edit the file `/etc/docker/daemon.json`. If the file doesn't exist or empty, add the content like this:
   ```json
   {
      "default-runtime": "nvidia",
      "runtimes": {
         "nvidia": {
            "path": "nvidia-container-runtime",
            "runtimeArgs": []
         }
      }
   }
   ```
   If the file contains some other runtime, add the nvidia runtime only and set it as default.
Everything is mentioned in [this post](https://github.com/NVIDIA/nvidia-docker/wiki/Driver-containers-(Beta)#quickstart) 

## How to install:
### 1. Labeling nodes:
Not all of the nodes contain GPUs or contain the right type of GPU, so this step will prevent K8s to schedule unwanted workload on the unsupported devices.

List all nodes
```bash
kubectl get all --show-labels
```
Eg: (the `LABELS` section is emmited)
```
NAME      STATUS   ROLES                      AGE    VERSION    LABELS
gpu1      Ready    worker                     34d    v1.14.5    key1=value1,key2=value2
gpu2      Ready    worker                     18d    v1.14.5    key1=value1,key2=value2
gpu3      Ready    worker                     18d    v1.14.5    key1=value1,key2=value2
storage   Ready    worker                     39d    v1.14.5    key1=value1,key2=value2
cp        Ready    controlplane,etcd          18d    v1.14.5    key1=value1,key2=value2
```
Label the node you want:
There's two kind of key-value pair you need to assign to the node:
   - `server-type=render` to mark the node as a GPU compatible. (required)
   - `driver-type=container` if you want to install driver in a container. If not, skip this label.
   
Assign the label by this command
```bash
kubectl label nodes <your-node-name> <your-key>=<your-value>
```
Eg:
```bash
kubectl label nodes gpu1 driver-type=container
kubectl label nodes gpu1 server-type=render
```
### 2. Install NVIDIA Driver:

__REMEMBER: NEVER INSTALL TWO TYPE OF DRIVER IN AN NODE, THIS WILL CRASH THE DRIVER WHICH IS INSTALLED LATER AND MAY MAKE THE K8s SCHEDULER CONFUSED AND MAKE THE NODE CRASHED__

   * Case one, use containerized driver:
      Simply enter
      ```bash
         # Ubuntu 16.04
         kubectl create -f https://raw.githubusercontent.com/h3d-haivq/nvidia-driver-kubernetes-yaml/master/nvidia-driver-ubuntu1604.yaml

         # Ubuntu 18.04
         kubectl create -f https://raw.githubusercontent.com/h3d-haivq/nvidia-driver-kubernetes-yaml/master/nvidia-driver-ubuntu1804.yaml
      ```

   * Case two, Use driver installed in the machine:
      Just install it like normal. Get the .RUN or .DEB file from the NVIDIA web page

### 3. Install NVIDIA device plugin
Simply enter
```bash
kubectl create -f https://raw.githubusercontent.com/h3d-haivq/nvidia-driver-kubernetes-yaml/master/nvidia-device-plugin.yaml
```

## How to make use of the NVIDIA device plugin:
This plugin will make the K8s scheduler automatically schedules your workload and serves correct amount of GPU you want by just some easy config. It's achived by adding `nvidia.com/gpu: <number-of-gpus>` in the `spec.containers.resources` of the YAML.

Eg:
```yml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  containers:
    - name: cuda-container
      image: nvidia/cuda:10.1-runtime
      resources:
        limits:
          nvidia.com/gpu: 2 # requesting 2 GPUs
```
In this case, any pod created by this YAML will consumes _no more than_ 2 GPUs, sperately from each other.

If you want your workload to use _exactly_ 2 GPUs, modify the `spec` section of the YAML above like this:
```yml
...
spec:
  containers:
    - name: cuda-container
      image: nvidia/cuda:10.1-runtime
      resources:
        requests:
          nvidia.com/gpu: 2
        limits:
          nvidia.com/gpu: 2
```

You can still use `NVIDIA_VISIBLE_DEVICES` variable if you want, which is kinda useless now.
