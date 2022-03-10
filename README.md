Setup and Deploy a Disconnected internal Registry

Install podman program

$ dnf install podman -y
Set up credentials for our internal registry

$ # Create registry root directory
$ mkdir -p /opt/registry/{auth,certs,data}

$ # Create certificate for our internal registry
$ openssl req \
  -newkey rsa:4096 \
  -nodes \
  -sha256 \
  -keyout /opt/registry/certs/registry.key \
  -x509 \
  -days 3650 \
  -out /opt/registry/certs/registry.crt \
  -addext "subjectAltName = DNS:registry.clus3a.t5g.lab.eng.bos.redhat.com" \
  -subj "/C=US/ST=Texas/L=Austin/O=Red Hat/OU=infra/CN=registry.clus3a.t5g.lab.eng.bos.redhat.com"

$ # Load the registry.crt into our host ca-trust bundle,
$ # in order to trust it as a CA
$ cp \
  /opt/registry/certs/registry.crt /etc/pki/ca-trust/source/anchors/
$ update-ca-trust extract

$ # Create the htpasswd file with the user password dummy 
$ # that will be the authentication needed on your Pull 
$ # Secret to access the registry
$ htpasswd -bBc /opt/registry/auth/htpasswd dummy dummy
Create internal registry configuration

$ mkdir -p /opt/registry/conf/
$ cat << EOF > /opt/registry/conf/config.yml
version: 0.1
log:
  fields:
    service: registry
storage:
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
compatibility:
  schema1:
    enabled: true
EOF

$ # NOTE: One of the most important parts it's the scheme 
$ # compatibility, without that, the mirroring process will not work.
$ tail -3 /opt/registry/conf/config.yml
compatibility:
  schema1:
    enabled: true
Start up the internal registry

$ # Login your docker account
$  podman login docker.io -u <your_id> 

$ # Create our podman registry container to host the OCP and OLM
$ # Container Images
$ podman run \
    --name registry \
    -d \
    --net host \
    -e REGISTRY_AUTH=htpasswd \
    -e REGISTRY_AUTH_HTPASSWD_REALM=Registry \
    -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
    -e REGISTRY_HTTP_SECRET=YourOwnLongRandomSecret \
    -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry.crt \
    -e REGISTRY_HTTP_TLS_KEY=/certs/registry.key \
    -e REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY=/registry \
    -v /opt/registry/auth:/auth:z \
    -v /opt/registry/certs:/certs:z \
    -v /opt/registry/data:/registry:z \
    -v /opt/registry/conf/config.yml:/etc/docker/registry/config.yml:z \
    registry:2.7

$ # Ensure we have the Firewall opened
$ firewall-cmd --add-port 5000/tcp --permanent
$ firewall-cmd --reload
$ firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: ens3
  sources:
  services: cockpit dhcpv6-client http ssh
  ports: 5000/tcp
  protocols:
  forward: no
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
Check the internal registry

$ # To check that the registry it's up and running,
$ # we need to write down the pull_secret.json
$ cat << EOF > pull-secret.json
{
  "auths": {
    "registry.clus3a.t5g.lab.eng.bos.redhat.com:5000": {
      "auth": "ZHVtbXk6ZHVtbXk="
    }
  }
}
EOF

$ # Install skopeo tool
$ dnf install skopeo -y

$ # Let's try to mirror an image manually using skopeo
$  skopeo copy \
  --authfile pull-secret.json \
  --all \
  docker://quay.io/quay/busybox:latest docker://registry.clus3a.t5g.lab.eng.bos.redhat.com:5000/test/busybox:latest

$ # Let's Check out Internal Registry
$ podman search --authfile pull-secret.json registry.clus3a.t5g.lab.eng.bos.redhat.com:5000/test/
INDEX            NAME                                                          DESCRIPTION  STARS       OFFICIAL    AUTOMATED
redhat.com:5000  registry.clus3a.t5g.lab.eng.bos.redhat.com:5000/test/busybox               0

$ podman exec -it registry ls -l /registry/docker/registry/v2/repositories
total 0
drwxr-xr-x    3 root     root            21 Nov 18 12:24 test

$ podman exec -it registry ls -l /registry/docker/registry/v2/repositories/test
total 0
drwxr-xr-x    5 root     root            55 Nov 18 12:24 busybox
Mirror an Openshift Container Platform Release

Set up credentials for our internal registry

Downloaded the pull secret from the Pull Secret page on the Red Hat OpenShift Cluster Manager site.
 


And modified it to include authentication to our mirror repository.

$ cat pull-secret.json
{
  "auths": {
    "registry.clus3a.t5g.lab.eng.bos.redhat.com:5000": {
      "auth": "ZHVtbXk6ZHVtbXk="
    }
    "cloud.openshift.com": {
      "auth": "b3BlbnNoaWZ0................23pjf9==",
      "email": "johndoe@redhat.com"
    },
    "quay.io": {
      "auth": "b3BlbnNoaWZ0L...............2p3idm==",
      "email": "johndoe@redhat.com"
    },
    "registry.connect.redhat.com": {
      "auth": "fHVoYy1wb2ad.................fsdkn==",
      "email": "johndoe@redhat.com"
    },
    "registry.redhat.io": {
      "auth": "fHVoYy1wb2a.................aslsm9==",
      "email": "johndoe@redhat.com"
    }
  }
}

Install libvirt library

$ # Install libvirt for openshift-baremetal-installer to work
$ dnf install libvirt -y
Mirroring the OpenShift Container Platform image repository

Review the OpenShift Container Platform downloads page to determine the version of OpenShift Container Platform that you want to install.

Then, determine the corresponding tag on the Repository Tags page.

$ # For this demo serie we are using:
$ export OCP_RELEASE=4.9.7

$ # Local registry name and host port
$ export LOCAL_REGISTRY=registry.clus3a.t5g.lab.eng.bos.redhat.com:5000

$ # Local repository name
$ export LOCAL_REPOSITORY=ocp4

$ # Name of the repository to mirror
$ export PRODUCT_REPO=openshift-release-dev

$ # Complete path to our registry pull secret
$ export LOCAL_SECRET_JSON=/opt/pull-secret.json

$ # Release mirror
$ export RELEASE_NAME=ocp-release

$ # Type of architecture for our server
$ export ARCHITECTURE=x86_64

At this point, two possibilities exist:

    1. Your mirror host has access to Internet (i.e.: an extra NIC connected to Internet)
    2. Your mirror host has not access to Internet
Your mirror host has access to the internet

$ # If the local container registry is connected to the mirror host, $ # take the following actions
$ oc adm release mirror \
-a ${LOCAL_SECRET_JSON} \
--from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE} \
--to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} \       --to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE}

. . .

sha256:65369b6ccff0f3dcdaa8685ce4d765b8eba0438a506dad5d4221770a6ac1960a registry.clus3a.t5g.lab.eng.bos.redhat.com:5000/ocp4:4.9.7-x86_64-driver-toolkit
info: Mirroring completed in 2m4.57s (81.35MB/s)

Success
Update image:  registry.clus3a.t5g.lab.eng.bos.redhat.com:5000/ocp4:4.9.7-x86_64
Mirror prefix: registry.clus3a.t5g.lab.eng.bos.redhat.com:5000/ocp4
Mirror prefix: registry.clus3a.t5g.lab.eng.bos.redhat.com:5000/ocp4:4.9.7-x86_64

To use the new mirrored repository to install, add the following section to the install-config.yaml:

imageContentSources:
- mirrors:
  - registry.clus3a.t5g.lab.eng.bos.redhat.com:5000/ocp4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - registry.clus3a.t5g.lab.eng.bos.redhat.com:5000/ocp4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev


To use the new mirrored repository for upgrades, use the following to create an ImageContentSourcePolicy:

apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: example
spec:
  repositoryDigestMirrors:
  - mirrors:
    - registry.clus3a.t5g.lab.eng.bos.redhat.com:5000/ocp4
    source: quay.io/openshift-release-dev/ocp-release
  - mirrors:
    - registry.clus3a.t5g.lab.eng.bos.redhat.com:5000/ocp4
    source: quay.io/openshift-release-dev/ocp-v4.0-art-dev

$ # Record the entire imageContentSources section from the output
$ # of the previous command. The information about your mirrors is
$ # unique to our mirrored repository, and we must add the
$ # imageContentSources section to the install-config.yaml file
$ # during installation.


$ # Create the installation program that is based on the content
$ # that we mirrored
$ oc adm release extract \
-a ${LOCAL_SECRET_JSON} \
--command=openshift-baremetal-install \
"${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE}"

$ # Ensure that we use the correct images for the version of
$ # OpenShift Container Platform that we selected
$ openshift-baremetal-install version

$ # Check new added repository
$ podman search --authfile pull-secret.json registry.clus3a.t5g.lab.eng.bos.redhat.com:5000/ocp4
$ ll registry/data/docker/registry/v2/repositories
$ ll registry/data/docker/registry/v2/repositories/ocp4/
Your mirror host has not access to the internet1

$ # Connect the removable media to a system that is connected
$ # to the internet.
$ export REMOVABLE_MEDIA_PATH=/opt/removable_media_folder

$ # Review the images and configuration manifests to mirror
$ oc adm release mirror -a ${LOCAL_SECRET_JSON} \ --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE} \
--to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} \ --to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE} --dry-run

. . .

Success
Update image:  registry.clus3a.t5g.lab.eng.bos.redhat.com:5000/ocp4:4.9.7-x86_64
Mirror prefix: registry.clus3a.t5g.lab.eng.bos.redhat.com:5000/ocp4
Mirror prefix: registry.clus3a.t5g.lab.eng.bos.redhat.com:5000/ocp4:4.9.7-x86_64

To use the new mirrored repository to install, add the following section to the install-config.yaml:

imageContentSources:
- mirrors:
  - registry.clus3a.t5g.lab.eng.bos.redhat.com:5000/ocp4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - registry.clus3a.t5g.lab.eng.bos.redhat.com:5000/ocp4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev


To use the new mirrored repository for upgrades, use the following to create an ImageContentSourcePolicy:

apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: example
spec:
  repositoryDigestMirrors:
  - mirrors:
    - registry.clus3a.t5g.lab.eng.bos.redhat.com:5000/ocp4
    source: quay.io/openshift-release-dev/ocp-release
  - mirrors:
    - registry.clus3a.t5g.lab.eng.bos.redhat.com:5000/ocp4
    source: quay.io/openshift-release-dev/ocp-v4.0-art-dev

$ # Record the entire imageContentSources section from the output
$ # of the previous command. The information about your mirrors is
$ # unique to our mirrored repository, and we must add the
$ # imageContentSources section to the install-config.yaml file
$ # during installation.

$ # Mirror the images to a directory on the removable media
$ oc adm release mirror \
-a ${LOCAL_SECRET_JSON} \
--to-dir=${REMOVABLE_MEDIA_PATH}/mirror \
quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE}

. . .

To upload local images to a registry, run:

    oc image mirror --from-dir=/opt/removable_media_folder/mirror 'file://openshift/release:4.9.7-x86_64*' REGISTRY/REPOSITORY


$ # Take the media to the restricted network environment and upload
$ # the images to the local container registry.
$ # NOTE: For REMOVABLE_MEDIA_PATH, we must use the same path that
$ # we specified when we mirrored the images
$ $ oc image mirror \
-a ${LOCAL_SECRET_JSON} \
--from-dir=${REMOVABLE_MEDIA_PATH}/mirror "file://openshift/release:${OCP_RELEASE}*" ${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}

$ # Create the installation program that is based on the content
$ # that we mirrored
$ oc adm release extract \
-a ${LOCAL_SECRET_JSON} \
--command=openshift-baremetal-install \
"${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}"

$ # Ensure that we use the correct images for the version of
$ # OpenShift Container Platform that we selected
$ openshift-baremetal-install version

$ # Check new added repository
$ podman search --authfile pull-secret.json registry.clus3a.t5g.lab.eng.bos.redhat.com:5000/ocp4
$ ll registry/data/docker/registry/v2/repositories
$ ll registry/data/docker/registry/v2/repositories/ocp4/
