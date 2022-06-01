# Setting up a self-signed private local registry with auth

## Setup

The following setup assumes:
- A secure local network. While the registry has been secured with basic Bcrypt authentication, steps should still be taken to make sure the network is secure.
- All variables in the format `$VARIABLE` be replaced with values based on the environment
- A registry container exposed on port `5000`, with a host address of `$LOCAL_IP`
- A balenaOS client with an mDNS address of `balena.local`

###  Generate certs on server

```bash
openssl req \
    -newkey rsa:4096 \
    -x509 \
    -sha256 \
    -days 365 \
    -nodes \
    -out certs/domain.crt \
    -keyout certs/domain.key \
    -addext "subjectAltName = IP:$LOCAL_IP"
```

These self-signed certs should be placed on the machine that will be hosting the `registry` container.

Every time the certificate is replaced, the `registry` container will need to be restarted to pick up the cert changes.

### Make Docker daemons aware of certificate

From the server filesystem, for development mode balena devices:
```bash
mkdir -p /etc/docker/certs.d/$LOCAL_IP:5000 && \
    scp -P 22222 certs/domain.crt root@balena.local:/etc/docker/certs.d/$LOCAL_IP:5000
```

From a container's filesystem, for production mode balena devices:
```bash
mkdir -p /etc/docker/certs.d/$LOCAL_IP:5000 && \
    balena cp $(balena ps -aq -f name=$CONTAINER):/path/to/crt/in/container /etc/docker/certs.d/$LOCAL_IP:5000
```

The process above of copying the cert to `/etc/docker/certs.d/$LOCAL_IP:5000` must be repeated for any Docker daemon that needs to talk with the local registry.

You don't need to restart the Docker daemon.

### Push an image to the registry

```bash
# Login to local registry
docker login $LOCAL_IP:5000 -u testuser -p testpassword
# Pull image on server host
docker pull $IMAGE
# Tag image with local registry address
docker tag $IMAGE $LOCAL_IP:5000/$IMAGE
# Push image from server host to local registry
docker push $LOCAL_IP:5000/$IMAGE
```

### Pull an image from the registry to a balena client

```bash
# Login to local registry
balena login $LOCAL_IP:5000 -u testuser -p testpassword
# Pull image from local registry
balena pull $LOCAL_IP:5000/$IMAGE
# Tag image in a way the Supervisor recognizes
balena tag $LOCAL_IP:5000/$IMAGE $IMAGE
# Remove local registry tag to ensure image is cleaned up by Supervisor after use
balena rmi $LOCAL_IP:5000/$IMAGE
```

### Tips
- Creating the client `certs.d` directory and copying in the `.crt` file may be automated by piping a script into `balena ssh`, allowing this change to be performed on a large fleet. [ssh-uuid](https://github.com/pdcastro/ssh-uuid) is an example of a tool that may be used to achieve this more easily.

- When `curl`ing or otherwise making a request the local registry, it may be necessary to pass the certificate file to avoid self-signed errors:

    ```bash
    # List images in repository
    curl --cacert /path/to/crt -u testuser:testpassword -v -X GET https://$LOCAL_IP:5000/v2/_catalog
    ```

- As local IP assignment on different networks may vary, it's recommended to advertise the registry server's IP using mDNS so that various registry local addresses may be uniformly resolved from `registry.local` or another address of your choosing. This makes it easier to interface with the registry in scripts. An example of a containerized app that publishes a device's `.local` address can be found [here](https://github.com/balena-io-playground/balena-avahi-dbus) when the host OS includes a system Avahi daemon, or [here](https://github.com/balena-io-playground/balena-mdns-service) when the host OS doesn't include Avahi. With minimal adaptations, either can be made to work outside of balena.

- Some versions of Docker require the certificate to be trusted at the OS level. See [here](https://docs.docker.com/registry/insecure/#docker-still-complains-about-the-certificate-when-using-authentication) for details.
