# Services: NFS Sever

## Index

- [Summary](#summary)
- [Diagram](#diagram)

## Summary

This deployment will set up NFS Server on `rpi-k8s-node-2`. In the near future, I want to add a new node on the K8s cluster and leave the `rpi-k8s-node-2` as a shared storage only.

_The NFS kernel server is currently the recommended NFS server for use with Linux, featuring features such as NFSv3 and NFSv4, Kerberos support via GSS, and much more_

**Obs:** Make sure the package `nfs-common` is installed on all nodes. This pkg is already added to the **common** role

## Diagram

![nfs-diagram](../../../img/nfs-diagram.png)
