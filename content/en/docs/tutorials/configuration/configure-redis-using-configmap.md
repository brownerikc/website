This tutorial guides you through configuring Redis using a ConfigMap. This will allow you to set up a Redis pod without a hard-coded configuration, then apply the configuration with a ConfigMap.

## Objectives
In this tutorial, you will learn:

* How to create a ConfigMap with Redis configuration values.
* How to create a Redis pod that mounts and uses the created ConfigMap.
* How to verify that the configuration was correctly applied.

## Prerequisites

You must have a Kubernetes cluster, and the kubectl command-line tool must be configured to communicate with your cluster. We recommend running this tutorial on a cluster with at least two nodes that are not acting as control plane hosts. If you do not already have a cluster, you can create one by using [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) or you can use one of these Kubernetes playgrounds:

* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)
* [Play with Kubernetes](https://labs.play-with-k8s.com/)

The commands in this tutorial work with kubectl version 1.14 and above. To check the version, enter the command ```kubectl version```.

## 1. Create a Config Map

Create a ConfigMap with an empty configuration block using the following command:

```shell
cat <<EOF >./example-redis-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-redis-config
data:
  redis-config: ""
EOF
```

This creates a ConfigMap file named `example-redis-config.yaml`. 

Apply this ConfigMap and a Redis pod manifest using the following commands:

```shell
kubectl apply -f example-redis-config.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
```

If you examine the contents of the Redis pod manifest, you will notice the following:

* A volume named `config` is created by `spec.volumes[1]`.
* The `key` and `path` under `spec.volumes[1].configMap.items[0]` exposes the `redis-config` key from the ConfigMap as a file named `redis.conf` on the `config` volume.
* The `config` volume is mounted at `/redis-master` by `spec.containers[0].volumeMounts[1]`.

This means that the configuration data in the ConfigMap is applied to the Redis configuration at `/redis-master/redis.conf` inside the pod.

The pod manifest should look similar to this:

```apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: redis:5.0.4
    command:
      - redis-server
      - "/redis-master/redis.conf"
    env:
    - name: MASTER
      value: "true"
    ports:
    - containerPort: 6379
    resources:
      limits:
        cpu: "0.1"
    volumeMounts:
    - mountPath: /redis-master-data
      name: data
    - mountPath: /redis-master
      name: config
  volumes:
    - name: data
      emptyDir: {}
    - name: config
      configMap:
        name: example-redis-config
        items:
        - key: redis-config
          path: redis.conf
```
		  
## 2. Validate the Configuration

Examine the created objects using the `kubectkl get` command:

```shell
kubectl get pod/redis configmap/example-redis-config 
```

You should see an output similar to the following:

```
NAME        READY   STATUS    RESTARTS   AGE
pod/redis   1/1     Running   0          8s

NAME                             DATA   AGE
configmap/example-redis-config   1      14s
```

Next, examine the objects using the `kubectkl describe` command for additional information:

```shell
kubectl describe configmap/example-redis-config
```

You should see an empty `redis-config` key. This is because we left the `redis-config` key in the ConfigMap empty. The output should look like this:

```shell
Name:         example-redis-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
redis-config:
```

## 3. Obtain Initial Configuration Values

Now let's get some of the initial configuration values of the Redis pod. These should change once you apply the ConfigMap to the pod.

Check the initial configuration using the `kubectl exec` and `redis-cli` commands:

```shell
kubectl exec -it redis -- redis-cli
```

Get the `maxmemory` value using the `CONFIG GET` command:

```shell
127.0.0.1:6379> CONFIG GET maxmemory
```

This should have a default value of 0:

```shell
1) "maxmemory"
2) "0"
```

Get the `maxmemory-policy` value using the `CONFIG GET` command:

```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

This should also have a default value of `noeviction`:

```shell
1) "maxmemory-policy"
2) "noeviction"
```

Make note of these values. These will change when you apply the configuration from the ConfigMap.

## 4. Apply Configuration Values Using the ConfigMap

Now let's add some configuration values to replace the default values you just checked. Update the `example-redis-config` ConfigMap to specify a `maxmemory` value of `2mb` and a `maxmemory-policy` value of `allkeys-lru`. The ConfigMap should look like this:

```apiVersion: v1
kind: ConfigMap
metadata:
  name: example-redis-config
data:
  redis-config: |
    maxmemory 2mb
    maxmemory-policy allkeys-lru
```

Apply the updated ConfigMap using the `kubectl apply` command:

```shell
kubectl apply -f example-redis-config.yaml
```

Confirm that the ConfigMap was updated using the `kubectl describe` command:

```shell
kubectl describe configmap/example-redis-config
```

You should see the configuration values you just added in `redis-config`:

```shell
Name:         example-redis-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
redis-config:
----
maxmemory 2mb
maxmemory-policy allkeys-lru
```

The ConfigMap now contains these new configurations. However, if you check the Redis pod configuration, you will notice that it's still using default configuations. Enter the Redis CLI by running the following command again:

```shell
kubectl exec -it redis -- redis-cli
```

Check `maxmemory` using the `CONFIG GET` command:

```shell
127.0.0.1:6379> CONFIG GET maxmemory
```

It remains the default value of 0:

```shell
1) "maxmemory"
2) "0"
```

Next, check `maxmemory-policy` using the `CONFIG GET` command:

```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

It also remains at the default value of `noeviction`:

```shell
1) "maxmemory-policy"
2) "noeviction"
```

To apply the new configuration values to the Redis pod, you must restart it.

## 5. Restart the Redis Pod

To restart the Redis pod, delete it and recreate if with the `kubectl apply` command:

```shell
kubectl delete pod redis
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
```

Enter the Redis CLI again using the following command:

```shell
kubectl exec -it redis -- redis-cli
```

Check `maxmemory` using the `CONFIG GET` command:

```shell
127.0.0.1:6379> CONFIG GET maxmemory
```

You should now see the updated value of 2097152:

```shell
1) "maxmemory"
2) "2097152"
```

Check the `maxmemory-policy` using the `CONFIG GET` command:

```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

You should also see the updated value of `allkeys-lru`:

```shell
1) "maxmemory-policy"
2) "allkeys-lru"
```

## 6. Clean Up Your Environment

Once you are done, you can delete the created resources using the `kubectl delete` command:

```shell
kubectl delete pod/redis configmap/example-redis-config
```

## What's Next

* Learn more about [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/).
* See more [ConfigMap examples](/docs/tasks/configure-pod-container/configure-pod-configmap/).
