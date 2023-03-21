# buildah/podman rootless-demo with micro containers
A demonstration of how you can use podman and buildah to create containers without root, when using ubi-micro containers.

This demo was created using Fedora 37.

# Installing prerequisites
Installing podman and buildah.
```
dnf install podman buildah
```

# Creating a container a tiny container
The container we will build is based on Red Hat's Universal Base Image and the special micro format. Micro containers do not come with the normal package managers, something we want to avoid anyways due to security risks. UBI-micro containers are small, as an example, [ubi9/ubi-micro](https://catalog.redhat.com/software/containers/ubi9/ubi-micro/615bdf943f6014fa45ae1b58) is only 7.3 MB compressed.

## The challenge and the solution
To build a container without package managers we instead copy files into a rootfs directory which buildah creates the container from.
The challenge is that the buildah mount function, used to create the rootfs directory is not rootless. In order to make it rootless, we create a build script, which does the building and we then use `buildah unshare` or `podman unshare` on that script. That launches the script into a namespace which makes it appear that we are root, when we are not.

## Building the container
1. To build a container which runs Apache web server, copy below script and name it rootless-rules.sh.
```
#!/bin/sh

# Fetch container and mount it
ctr=$(buildah from registry.redhat.io/ubi9/ubi-micro)
mnt=$(buildah mount $ctr)

# Install Apache and configure the landing page
dnf -y install --installroot $mnt httpd
echo "Never use root to build containers" >$mnt/var/www/html/index.html

# Cleanup
rm -rf $mnt/var/cache
rm $mnt/var/log/* 2>/dev/null

# Set ENTRYPOINT and EXPOSE port.
buildah config --entrypoint '/usr/sbin/httpd -D FOREGROUND' --port 80 $ctr

# Commit it to local
buildah commit $ctr dev/httpd-ubi9-micro
```
2. Now run `buildah unshare` or `podman unshare` on the script.
```
podman unshare rootless-rules.sh
# or
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

## Read more
* To learn more about podman/buildah unshare, checkout the [podman-unshare(1) man page](https://docs.podman.io/en/latest/markdown/podman-unshare.1.html).
* To learn more about rootless builds, [checkout Dan Walsh's blogpost, here.](https://opensource.com/article/19/3/tips-tricks-rootless-buildah)
* [Red Hat UBI 9 container](https://catalog.redhat.com/software/containers/ubi9/ubi-micro/615bdf943f6014fa45ae1b58)
