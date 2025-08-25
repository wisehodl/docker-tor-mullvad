# Docker Tor Hidden Service with Mullvad VPN

This is a Docker Compose application that creates a Tor hidden service (.onion
site) that is routed through a Mullvad VPN connection. This configuration
ensures that all Tor traffic is encrypted and routed through Mullvad's VPN
before entering the Tor network, providing an additional layer of privacy and
security.

You may not need such complexity in your own application, so please consider
your privacy and security needs accordingly.

## Table of Contents

- [Network Architecture](#network-architecture)
- [Application Architecture](#application-architecture)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
    - [1. Create Project Structure](#1-create-project-structure)
    - [2. Obtain Mullvad Credentials](#2-obtain-mullvad-credentials)
    - [3. Create Secrets Files](#3-create-secrets-files)
    - [4. Configure Mullvad Server Selection](#4-configure-mullvad-server-selection)
    - [5. Start the Service](#5-start-the-service)
- [Verifying Tor and VPN Routing](#verifying-tor-and-vpn-routing)
- [Accessing the Service](#accessing-the-service)
- [Troubleshooting](#troubleshooting)


## Network Architecture

```txt
User with Tor Browser
        |
        | Tor circuit request to .onion address
        | (3-hop encrypted onion routing)
        v
   Tor Network
        |
        | Exit node connects to what appears to be
        | Mullvad's public IP (final hop decrypted)
        v
Mullvad Infrastructure
        |
        | WireGuard encrypted tunnel carrying
        | Tor traffic to VPN client
        v
  Docker Application
  ┌─────────────────────────────────────────────────┐
  │  Shared VPN Network Namespace                   │
  │                                                 │
  │  Gluetun VPN Client                             │
  │         |                                       │
  │         | Decrypts WireGuard tunnel,            │
  │         | forwards to local Tor daemon          │
  │         v                                       │
  │  Tor Daemon (:9050)                             │
  │         |                                       │
  │         | SOCKS proxy forwards HTTP             │
  │         | requests to web application           │
  │         v                                       │
  │  Web Service (:8080)                            │
  │         |                                       │
  │         | Serves web server                     │
  │         | HTML response                         │
  │         |                                       │
  └─────────────────────────────────────────────────┘
```

## Application Architecture

This setup consists of three services:

1. **VPN Service**: Establishes connection to Mullvad servers
2. **Tor Service**: Runs a Tor daemon hidden service, routing through the VPN
3. **Web Service**: Serves a website through Tor and the VPN

All network traffic in and out of the services has to flow through the VPN,
ensuring the real server IP doesn't leak (via the docker services, at least).

## Prerequisites

- Docker and Docker Compose installed
- An active Mullvad VPN account
- Basic understanding of Tor hidden services and VPN routing

## Setup

### 1. Create Project Structure

1. **Clone or create the project structure**

```bash
git clone https://github.com/wisehodl/docker-tor-vpn.git
```

or manually:

```bash
mkdir docker-tor-vpn && cd docker-tor-vpn

# Create project files
```

The next steps involve setting up the required Mullvad secrets to allow a VPN
connection.

### 2. Obtain Mullvad Credentials

To run traffic through Mullvad VPN, you will need two pieces of information:

1. A WireGuard Private Key
2. A Mullvad server IP address

To obtain this information, download a WireGuard configuration file from:

https://mullvad.net/en/account/wireguard-config

1. Click **"Generate key"** to create a new WireGuard key pair
2. Select your desired **country**, **city**, and **server**
3. Download the configuration file

The downloaded file will look something like this:

```conf
[Interface]
# Device: Dreary Fox
PrivateKey = jeaXocgIni0gDB18yYY8eaX/MUUqFZ4CKmyrShIVK+s=
Address = 10.164.97.26/32,cc02:dc63:b9be:482e:f4be:9ec5:5eb9:217/128
DNS = 10.18.185.135

[Peer]
PublicKey = Qm5yJioxhmpmBN/BIsBIdnBGPHXKmsiM+a3EaYK2MhI=
AllowedIPs = 0.0.0.0/0,::0/0
Endpoint = 172.29.49.138:51820
```

### 3. Create Secrets Files

Create the .secrets directory in your project root

```bash
mkdir -p .secrets
```

`.secrets/mullvad_private_key`

Extract the value after `PrivateKey =` from the WireGuard config:

```bash
echo "jeaXocgIni0gDB18yYY8eaX/MUUqFZ4CKmyrShIVK+s=" > .secrets/mullvad_private_key
```

`.secrets/mullvad_address`

Extract the IPv4 address and subnet mask on the  `Address =` line:

```bash
echo "10.164.97.26/32" > .secrets/mullvad_address
```

**Security Note**: These files should be considered secret, as they contain
sensitive credentials. Make sure they:

- Are never committed to version control (`echo .secrets >> .gitignore`)
- Have restricted file permissions (`chmod 600 -R .secrets`)
- Stored securely and backed up separately.

### 4. Configure Mullvad Server Selection

In `compose.yml`, update the `SERVER_HOSTNAMES` environment variable to match
your chosen Mullvad server (typically, the filename of the config file you
downloaded). This value typically looks like:

- `us-dal-wg-401` (Dallas, US)
- `se-sto-wg-001` (Stockholm, Sweden)
- `nl-ams-wg-201` (Amsterdam, Netherlands)
- `de-fra-wg-001` (Frankfurt, Germany)

### 5. Start the Service

Start the service with Docker Compose:

```bash
docker compose up -d
```

Check the status of the services with:

```bash
docker compose ps -a
```

And note whether all the services are running.

Monitor the processes with:

```bash
docker compose logs -f
```

## Verifying Tor and VPN Routing

As this is a secure and private application setup, you should take the proper
steps to verify that your service is indeed being routed through the Tor and
Mullvad networks.

You can do this by requesting your IP address from an external service in the
Web Service container and by checking whether your connection appears as a Tor
connection.

Enter a shell inside the Web Service container:

```bash
docker compose exec web sh
```

Check your IP address without Tor. This should be the Mullvad VPN endpoint in
the city you selected in your `compose.yml` file.

```bash
curl https://icanhazip.com
```

Then, check your IP address with Tor using `proxychains` (due to the network
configuration of this applications, `torsocks` will not work). This should be
the Tor exit node your connection is being routed through, and completely
different than the Mullvad endpoint address.

```bash
proxychains curl https://icanhazip.com
```

Lastly, check to see if external services view your connection as coming from
the Tor network.

```bash
proxychains curl -s https://check.torproject.org | grep -i congratulations
```

You should see the text: `Congratulations. This browser is configured to use
Tor.`

## Accessing the Service

This is a Tor hidden service, so you will need to install Tor Browser to access
it.

Obtain the `.onion` address of the service from the Tor service container:

```bash
docker compose exec tor cat /var/lib/tor/hidden_service/hostname
```

This will return a `.onion` address. Copy and paste this value into Tor
Browser. If everything is configured properly and running, you should connect
to your web service.

If you see the content from the web server, then congratulations! You've set up
a Tor hidden service routed through a Mullvad VPN running in Docker Compose.

## Troubleshooting

This is a complex and uncommon setup, so some troubleshooting may be required.
These are some common issues you may encounter. Always check the docker logs to
look for error messages.

**VPN Connection Issues:**

- Verify your Mullvad account is active and its servers are available
- Check that the secrets files contain the correct values and don't have any
extra spaces or lines around the text
- Make sure the selected server hostname matches Mullvad's naming convention.

**Tor Service Issues:**

- The hidden services directory within the container MUST NOT be owned by
`root` AND have restricted permissions (in this configuration, it is owned by
the `tor` user and the directory has `700` permissions, while each file has
`600` permissions)
- Make sure you are not running another Tor service on the same machine,
causing port conflicts
- If the VPN is not active, the Tor connection will fail.

**Web Service Issues**

- Confirm that the web service is listening on `0.0.0.0:8080` and not
`localhost:8080`
- Verify there are no firewall rules or iptables blocking network traffic
between containers.
