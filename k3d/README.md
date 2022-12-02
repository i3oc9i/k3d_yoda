# Setup the k3d_yoda Kluster

Create the cluster
```
make k3d-up
```

add the following lines to `/etc/hosts`
```
127.1.0.1       yoda.local             # Load balancer
127.1.0.2       k3d.yoda.local         # Kubernetes API
127.1.0.3       registry.yoda.local    # Image registry
```

Destroy the cluster
```
make k3d-down
```
