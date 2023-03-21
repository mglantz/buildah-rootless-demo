# buildah-rootless-demo
A demonstration of how you can use buildah to create containers without root, when using ubi-micro containers.

This demo was created using Fedora 37.

# Installing prerequisites
Installing podman and buildah.
```
dnf install podman buildah
```

# Creating a container
The container is based on Red Hat's Universal Base Image and the special micro format. Micro containers do not come with the normal package managers, something we want to avoid anyways due to security risks.

## The challenge
To build container without package managers we instead copy files into a directory which buildah creates a container from.
The challenge is that the buildah mount function normally is not rootless. In order to make it rootless, we create a build script, which does the building and we then use `buildah unshare` on that script. That command will fool the system into thinking we are root, when we are not.

## Building the container
1. To build a container which runs Apache web server, copy below script and name it rootless-rules.sh.
```
#!/bin/sh

# Fetch container and mount it
ctr=$(buildah from registry.redhat.io/ubi9/ubi-micro)
mnt=$(buildah mount $ctr)

dnf -y install --installroot $mnt httpd
echo "Never use root to build containers" >$mnt/var/www/html/index.html

# Cleanup
rm -rf $mnt/var/cache
rm $mnt/var/log/* 2>/dev/null

# Set ENTRYPOINT
buildah config --entrypoint '/usr/sbin/httpd -D FOREGROUND' --port 80 $ctr

# Commit it to local
buildah commit $ctr dev/httpd-ubi9-micro
```
2. Now run `buildah unshare` on the script.
```
buildah unshare rootless-rules.sh
```

3. Validate that it got built.
```
podman images
```

4. Run the container
```
podman run --name microhttpd -d -p 8080:80 dev//httpd-ubi9-micro
```

5. Check so that the container is running.
```
podman ps -a
curl localhost:8080
```

Last command should return:
```
$ curl localhost:8080
Never use root to build containers
$
```
