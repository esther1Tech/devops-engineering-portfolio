# Docker Application Containerization

## Introduction

A Docker image is a versioned, read-only package containing an application and the files, runtime, libraries, and configuration defaults it needs to start. A container is a running instance of that image. Images are built once and can be run in development, test, and production environments, subject to the target platform and runtime requirements.

## 1. Why Build a Docker Image?

Building an image makes an application easier to package, test, distribute, deploy, and operate:

- **Consistent environments:** The application and its dependencies are packaged together, reducing differences between a developer's machine, CI, and production.
- **Repeatable delivery:** A pipeline can build and test an image, then promote the same artifact through environments rather than rebuilding the application differently at each stage.
- **Dependency isolation:** Each application runs with its own userspace dependencies, reducing conflicts with other applications on the host.
- **Portability:** The same image can run on Docker-compatible environments and container platforms, provided the target supports its CPU architecture and required operating-system features.
- **Scalability and recovery:** Orchestrators can start additional containers from an image, replace failed instances, and roll out new versions.
- **Traceability and rollback:** Tags and immutable image digests identify what was built and deployed, making releases easier to audit and previous versions easier to restore.
- **Automation:** CI/CD systems can build, scan, test, publish, and deploy images as repeatable pipeline steps.
- **Efficient distribution:** Images use reusable layers, so registries and hosts can avoid transferring layers they already have.

An image is not a virtual machine: it normally shares the host's operating-system kernel. It also should not contain environment-specific secrets or mutable application data.

## 2. Information to Collect Before Writing a Dockerfile

Gather these details before choosing a base image or defining build steps:

1. **Application and runtime**
   - Application type and language (for example, API, web frontend, worker, or model-serving service).
   - Supported runtime and version, operating-system requirements, CPU architecture, and any native/system libraries.
   - The command that starts the application and whether it needs initialization or a separate build step.

2. **Build and dependency process**
   - Build tool and exact build commands.
   - Dependency manifests and lockfiles, including how production-only dependencies are selected.
   - Files required at build time versus files required at runtime.
   - Whether compilation or asset generation allows a multi-stage build that keeps build tools out of the final image.

3. **Runtime interface**
   - Network ports the application listens on, and whether it must bind to `0.0.0.0` inside the container.
   - Required environment-variable names, safe defaults, and configuration sources. Document secret names or references, but never put secret values in the image.
   - Health/readiness checks, graceful shutdown behavior, signals, and expected process lifecycle.

4. **Data and storage**
   - Which paths, if any, need persistent or shared storage through volumes or external services.
   - Whether the application writes temporary files and where they should go.
   - Keep databases, user uploads, model artifacts, and other mutable state outside the image unless there is a specific reason to package immutable assets.

5. **Security and operations**
   - A trusted, supported base image and its update policy.
   - The least-privileged user the application can run as, required file permissions, and any capabilities it needs.
   - Image-size, startup-time, resource, vulnerability-scanning, and compliance requirements.
   - Logging destination (typically standard output/error) and any monitoring or telemetry expectations.

6. **Build context and delivery**
   - The smallest set of source files needed in the build context.
   - Files to exclude with `.dockerignore` (such as `.git`, local build output, caches, and local secret files).
   - Image naming, versioning/tagging policy, target registry, and how the pipeline identifies a release.
   - Required build arguments, target platforms, and any registry authentication needs. Credentials should be provided securely by the build system, not copied into image layers.

These inputs help produce a Dockerfile that is reproducible, secure, appropriately small, and suitable for its actual runtime environment.

## 3. Structure of a Docker Image

A Docker image consists of ordered filesystem layers and image configuration. Each Dockerfile instruction that changes the filesystem commonly contributes a layer; later layers build on earlier ones. Layers are content-addressed and can be shared across images.

```text
Application image
├── Image configuration and metadata
│   ├── Default command / entrypoint
│   ├── Default environment variables
│   ├── Working directory and user
│   ├── Exposed-port metadata and labels
│   └── Platform information (OS and architecture)
├── Application layer(s)
│   └── Application source, compiled output, or static assets
├── Dependency layer(s)
│   └── Runtime libraries and application dependencies
└── Base-image layers
    └── Minimal operating-system userspace and language runtime, as selected
```

The precise layers depend on the Dockerfile and builder; not every instruction creates a filesystem layer, and image configuration is distinct from filesystem content. At runtime, the container adds a writable layer on top of the image's read-only layers. Data that must survive container replacement should use an external service or persistent storage rather than relying on that writable layer.

### Example build flow

```text
Source code + dependency lockfile + Dockerfile + .dockerignore
                             |
                             v
                       Docker build
                             |
                             v
             Versioned image stored in a registry
                             |
                             v
         Containers started from that image in each environment
```

For a DevOps application, CI should build and test the image, scan it, and publish it with a release identifier. Deployment should refer to the tested image—preferably by digest for exact identity—while supplying environment-specific configuration and secrets at runtime.

For an MLOps service, the image can package the serving code, runtime, and required libraries. Model versions and large or frequently updated model artifacts are often managed separately in a model registry or object store and provided to the service at runtime, depending on the reproducibility and deployment design.
