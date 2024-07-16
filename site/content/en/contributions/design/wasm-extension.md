---
title: "Wasm OCI Image Support"
---

## Motivation
Envoy Gateway (EG) should support Wasm OCI image as a remote wasm code source.
This feature will allow users to deploy Wasm modules from OCI registries, such as Docker Hub, 
Google Container Registry, and Amazon Elastic Container Registry, to Envoy proxies managed by EG.
Deploying Wasm modules from OCI registries has several benefits:
* **Versioning**: Users can use the tag feature of the OCI image to manage the version of the Wasm module.
* **Security**: Users can use private registries to store the Wasm module.
* **Distribution**: Users can use the existing distribution mechanism of the OCI registry to distribute the Wasm module.

## Goals
* Define the system components needed to support Wasm OCI images as remote Wasm code sources.

## Architecture

![](/img/wasm-extension.png)

### Control Plane Wasm File Cache

Envoy lacks native OCI image support, therefore, EG needs to download Wasm modules from their original OIC registries, 
cache them locally in the file system, and serve them to Envoy over HTTP.

#### HTTP Code Source

For HTTP code source, we have two options: serve Wasm modules directly from their 
original HTTP URLs, or cache them in EG (as with OCI images). Caching both the HTTP Wasm modules and OCI 
images inside EG can make UI consistent, for example, sha256sum can be calculated on the EG side and made 
optional in the API. This will also make the Envoy proxy side more efficient as it won’t have to download the
Wasm module from the original URL every time, which can be slow.

#### Resource Consumption

Memory: Since we cache Wasm modules in the file system, we can optimize the memory usage 
of the cache and the HTTP server to avoid introducing too much memory consumption by this feature. 
For example, when receiving a Wasm pulling request form the Envoy, we can open the Wasm file and write 
the file content directly to response, and then close the file. There won’t be significant memory consumption involved. 

Disk: Though it's possible to mount a volume to the container for the Wasm file cache, the current implementation just
stores the Wasm files in the EG container’s file system. The disk space consumed by the cache is limited to 1GB by default, 
it can be made configurable in the future. 

#### Caching Mechanism

Cached files will be evicted based on LRU（Last recently used）algorithm. 
If the image’s tag is latest, then they will be updated periodically. The cache clean and update periods 
will be configurable.

#### Restrict Access to Private Images

* **Client Authn with mTLS:** To prevent unauthorized proxies from accessing the Wasm modules, the communication between 
  the Envoy and EG will be secured using mTLS.
* **User Authn with Registry Credentials:** To prevent unauthorized users from accessing the Wasm modules, the user who 
  creates the EEP must have the appropriate permissions to access the OCI registry. For example, if two users create EEPs
  in different namespaces (ns1, ns2) accessing the same OCI image, each must also create a unique secret with registry 
  credentials (secret1 for user1 in ns1, secret2 for user2 in ns2) and provide it in the EEP configuration. EG will validate
  the provided secret against the OCI registry before serving the Wasm module to the target HTTPRoute/Gateway of that EEP.
* **Unguessable Download URLs:** It's possible that users who have no permission to access a private OCI image could create
  an EnvoyPatchPolicy to bypass the EG permission check. For example, a user could create an EPP, inject a Wasm filter, 
  and put the download URL of a private OCI image in the Wasm filter configuration. To prevent this, we need to make the 
  download URL unguessable. The download URL will be generated by EG and will be a random string that is impossible to 
  guess. If a user can get the config dump the Envoy Proxy, they can still get the download URL. However, they won’t be 
  able to do so without the permission to access the config dump of Envoy Proxy, which is a more restricted permission (usually Admin role).

## Alternative Considered

### Inline Bytes

EG downloads the Wasm modules, caches them in memory, and pushes the Wasm code through xDS as inline bytes. 
This could inflate the xDS, potentially causing memory issues for EG and Envoy.

### Data Plane Agent
Mount Wasm modules in the local file system of the Envoy container. We’ll need an agent deployed in the 
same pod as the Envoy for this. It would be too expensive to implement this as we’ll need to intercept 
the xDS at the agent.

### Standalone Wasm HTTP Server
Deploying the Wasm HTTP server as a standalone service. While this has no obvious benefits, it increases 
operational costs.

### Wait for Envoy OCI image support
We could wait indefinitely for Envoy to support OCI imag as a remote wasm code source.