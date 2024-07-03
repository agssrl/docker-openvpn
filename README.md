# Differences from the original

- Updated Alpine base and packages
- Added DNS configuration for AdGuard
- Added to duplicate_cn
- Key size increased to 4096 bits
- AdGuard DNS integrated

# OpenVPN for Docker

OpenVPN server in a Docker container complete with an EasyRSA PKI CA.

#### Upstream Links

* Docker Registry @ [agssrl/docker-openvpn](https://hub.docker.com/r/agssrl/docker-openvpn)
* Docker Registry @ [nubacuk/docker-openvpn](https://hub.docker.com/r/nubacuk/docker-openvpn)
* Original GitHub @ [kylemanna/docker-openvpn](https://github.com/kylemanna/docker-openvpn)

## Quick Start for aarch64

* Pick a name for the `$OVPN_DATA` data volume container. It's recommended to
* use the `ovpn-data-` prefix to operate seamlessly with the reference systemd
* service. Users are encouraged to replace `example` with a descriptive name of their choosing.

  The architectures supported by this image are:

| Architecture | Available | Tag |
| :----: | :----: | ---- |
| x86-64 | ✅ | x86_64 |
| arm64 | ✅ | aarch64 |

  There are images for aarch64 (ARM) and x86_64.

      OVPN_DATA="ovpn-data"
      OVPN_PLATFORM="aarch64"

* Initialize the `$OVPN_DATA` container that will hold the configuration files and certificates. The container will prompt for a passphrase to protect the private key used by the newly generated certificate authority.

      docker volume create --name $OVPN_DATA
      docker run -v $OVPN_DATA:/etc/openvpn --rm agssrl/docker-openvpn:$OVPN_PLATFORM ovpn_genconfig -u udp://VPN.SERVERNAME.COM
      docker run -v $OVPN_DATA:/etc/openvpn --rm -it agssrl/docker-openvpn:$OVPN_PLATFORM ovpn_initpki

* Start the OpenVPN server process

      docker run -v $OVPN_DATA:/etc/openvpn -d -p 1194:1194/udp --cap-add=NET_ADMIN --restart=always --name openvpn agssrl/docker-openvpn:$OVPN_PLATFORM

* Generate a client certificate without a passphrase

      docker run -v $OVPN_DATA:/etc/openvpn --rm -it agssrl/docker-openvpn:$OVPN_PLATFORM easyrsa build-client-full CLIENTNAME nopass

* Retrieve the client configuration with embedded certificates

      docker run -v $OVPN_DATA:/etc/openvpn --rm agssrl/docker-openvpn:$OVPN_PLATFORM ovpn_getclient CLIENTNAME > CLIENTNAME.ovpn

## ERRORS

      modprobe: can't change directory to '/lib/modules': No such file or directory
      iptables v1.8.4 (legacy): can't initialize iptables table `nat': Table does not exist (do you need to insmod?)
      Perhaps iptables or your kernel needs to be upgraded.

Furthermore, on my system (CentOS / RockyLinux) I had to run the following on the host:

      sudo modprobe iptable_filter
      sudo modprobe iptable_nat

### More Reading

Miscellaneous write-ups for advanced configurations are available in the [docs](docs) folder.

### Systemd Init Scripts

A `systemd` init script is available to manage the OpenVPN container.
It will start the container on system boot, restart the container if it exits
unexpectedly, and pull updates from Docker Hub to keep itself up to date.

Please refer to the [systemd documentation](docs/systemd.md) to learn more.

### Docker Compose

If you prefer to use `docker-compose`, please refer to the [documentation](docs/docker-compose.md).

## Debugging Tips

* Create an environment variable with the name DEBUG and value of 1 to enable debug output (using "docker -e").

        docker run -v $OVPN_DATA:/etc/openvpn -p 1194:1194/udp --cap-add=NET_ADMIN -e DEBUG=1 agssrl/docker-openvpn:$OVPN_PLATFORM

* Test using a client that has OpenVPN installed correctly

        $ openvpn --config CLIENTNAME.ovpn

* Run through a barrage of debugging checks on the client if things don't just work

        $ ping 8.8.8.8    # checks connectivity without touching name resolution
        $ dig google.com  # won't use the search directives in resolv.conf
        $ nslookup google.com # will use search

* Consider setting up a [systemd service](/docs/systemd.md) for automatic start-up at boot time and restart in the event the OpenVPN daemon or Docker crashes.

## How Does It Work?

Initialize the volume container using the `agssrl/docker-openvpn:arm64` image with the included scripts to automatically generate:

- Diffie-Hellman parameters
- A private key
- A self-certificate matching the private key for the OpenVPN server
- An EasyRSA CA key and certificate
- A TLS auth key from HMAC security

The OpenVPN server is started with the default run cmd of `ovpn_run`.

The configuration is located in `/etc/openvpn`, and the Dockerfile declares that directory as a volume. This means that you can start another container with the `-v` argument and access the configuration. The volume also holds the PKI keys and certs so that it could be backed up.

To generate a client certificate, `agssrl/docker-openvpn:arm64` uses EasyRSA via the `easyrsa` command in the container's path. The `EASYRSA_*` environmental variables place the PKI CA under `/etc/openvpn/pki`.

Conveniently, `agssrl/docker-openvpn:arm64` comes with a script called `ovpn_getclient`, which dumps an inline OpenVPN client configuration file. This single file can then be given to a client for access to the VPN.

To enable Two-Factor Authentication for clients (a.k.a. OTP), see [this document](/docs/otp.md).

## OpenVPN Details

We use `tun` mode because it works on the widest range of devices. `Tap` mode, for instance, does not work on Android unless the device is rooted.

The topology used is `net30` because it works on the widest range of OS. `P2p`, for instance, does not work on Windows.

The UDP server uses `192.168.255.0/24` for dynamic clients by default.

The client profile specifies `redirect-gateway def1`, meaning that after
establishing the VPN connection, all traffic will go through the VPN. 
This might cause problems if you use local DNS recursors which are not
directly reachable since you will try to reach them through the VPN
and they might not answer to you. If that happens, use public DNS 
resolvers like those of Google (8.8.4.4 and 8.8.8.8) or OpenDNS
(208.67.222.222 and 208.67.220.220).

## Security Discussion

The Docker container runs its own EasyRSA PKI Certificate Authority. This was
chosen as a good way to compromise on security and convenience. The container
runs under the assumption that the OpenVPN container is running on a secure 
host, meaning that an adversary does not have access to the PKI files
under `/etc/openvpn/pki`. This is a fairly reasonable compromise because if an
adversary had access to these files, they could manipulate the
function of the OpenVPN server itself (sniff packets, create a new PKI CA, MITM 
packets, etc.).

* The certificate authority key is kept in the container by default for
* simplicity. It's highly recommended to secure the CA key with a
* passphrase to protect against filesystem compromise. A more secure system
* would put the EasyRSA PKI CA on an offline system (can use the same Docker
* image and the script [`ovpn_copy_server_files`](/docs/paranoid.md) to accomplish this).
* It would be impossible for an adversary to sign bad or forged certificates
* without first cracking the key's passphrase should the adversary have root
* access to the filesystem.
* The EasyRSA `build-client-full` command will generate and leave keys on the
* server, which could be compromised and the keys stolen. The generated keys need to be signed by the CA, which the user should hopefully configure with a passphrase as described above.
* Assuming the rest of the Docker container's filesystem is secure, TLS + PKI
* security should prevent any malicious host from using the VPN.

## Benefits of Running Inside a Docker Container

### The Entire Daemon and Dependencies are in the Docker Image

This means that it will function correctly (after Docker itself is set up) on
all Linux distributions such as Ubuntu, Arch, Debian, Fedora, etc. 
Furthermore, an old stable server can run a bleeding-edge OpenVPN server
without having to install or muck with library dependencies (i.e., run the latest
OpenVPN with the latest OpenSSL on Ubuntu 12.04 LTS).

### It Doesn't Stomp All Over the Server's Filesystem

Everything for the Docker container is contained in two images:
the ephemeral runtime image (agssrl/docker-openvpn:arm64) and the `$OVPN_DATA` data volume.
To remove it, simply remove the corresponding containers, `$OVPN_DATA` data volume, and Docker
image, and it's completely removed. This also makes it easier to run multiple servers since each 
lives in the bubble of the container (of course, multiple IPs or separate ports are needed to
communicate with the world).

### Some (arguable) Security Benefits

At the simplest level, compromising the container may prevent additional compromise of the server.
There are many arguments surrounding this, but the takeaway is that it certainly makes it more 
difficult to break out of the container. People are actively working on Linux containers to make 
this more of a guarantee in the future.

## Differences from jpetazzo/dockvpn

* No longer uses serveconfig to distribute the configuration via https
* Proper PKI support integrated into the image
* OpenVPN config files, PKI keys, and certs are stored on a storage volume for re-use across containers
* Addition of tls-auth for HMAC security

## Originally Tested On

* Docker hosts:
  * Server a [Oracle](https://cloud.oracle.com/compute/instances) Droplet with VM.Standard.A1.Flex (4CPU Ampere and 24GB RAM) running Oracle 8.
* Clients:
  * Android App OpenVPN Connect
  * OS X Monterey with Tunnelblick
  * ArchLinux OpenVPN pkg 2.3.4-1

## Setting Up Docker Hub Automatic Builds with Access Token

To configure automatic builds on Docker Hub using a secure access token, follow these detailed steps:

1. **Navigate to Your GitHub Repository:** Open the project you intend to set up for automatic builds.

2. **Access Repository Settings:** Click on `Settings` within your repository to access various project configuration options.

3. **Go to Secrets and Variables:** Navigate to the `Secrets and variables` section, then select `Actions`. This allows you to manage secrets used in your GitHub Actions workflows without exposing them in your code.

4. **Add a Secret for Docker Hub Authentication:**
   - `DOCKER_TOKEN`: Create this secret to store your Docker Hub access token. Generate an access token by going to your Docker Hub account settings, navigating to `Security`, and clicking on `New Access Token`. Make sure the token has appropriate permissions for pushing images to your repositories.

5. **Update Your GitHub Actions Workflow:**
   Add the following steps in your GitHub Actions workflow file to authenticate to Docker Hub using the token and push the image:

   ```yaml
   - name: Log in to Docker Hub
     uses: docker/login-action@v1
     with:
       username: ${{ github.actor }}
       password: ${{ secrets.DOCKER_TOKEN }}

   - name: Push to Docker Hub
     run: |
       docker build -t yourusername/yourimagename:$TAG .
       docker push yourusername/yourimagename:$TAG
