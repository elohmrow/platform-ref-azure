# Azure Reference Platform for Kubernetes + Data Services

This repository contains a reference Azure Platform
[Configuration](https://crossplane.io/docs/v1.6/getting-started/create-configuration.html)
for use as a starting point in [Upbound Cloud](https://upbound.io) or
[Upbound Universal Crossplane (UXP)](https://www.upbound.io/uxp/) to build,
run and operate your own internal cloud platform and offer a self-service
console and API to your internal teams. It provides platform APIs to provision
fully configured Azure AKS clusters, with secure networking, and stateful cloud
services (Azure Database for PostgreSQL) designed to securely connect to the nodes in each AKS cluster --
all composed using cloud service primitives from the [Crossplane Azure
Provider](https://doc.crds.dev/github.com/crossplane/provider-azure). App
deployments can securely connect to the infrastructure they need using secrets
distributed directly to the app namespace.

## Contents

* [Upbound Cloud](#upbound-cloud)
* [Build Your Own Internal Cloud Platform](#build-your-own-internal-cloud-platform)
* [Quick Start](#quick-start)
* [Platform Ops/SRE: Run your own internal cloud platform](#platform-opssre-run-your-own-internal-cloud-platform)
  * [App Dev/Ops: Consume the infrastructure you need using kubectl](#app-devops-consume-the-infrastructure-you-need-using-kubectl)
  * [APIs in this Configuration](#apis-in-this-configuration)
* [Customize for your Organization](#customize-for-your-organization)
* [Learn More](#learn-more)

## Upbound Cloud

![Upbound Overview](docs/media/upbound.png)

What if you could eliminate infrastructure bottlenecks, security pitfalls, and
deliver apps faster by providing your teams with self-service APIs that
encapsulate your best practices and security policies, so they can quickly
provision the infrastructure they need using a custom cloud console, `kubectl`,
or deployment pipelines and GitOps workflows -- all without writing code?

[Upbound Cloud](https://upbound.io) enables you to do just that, powered by the
open source [Upbound Universal Crossplane](https://www.upbound.io/uxp/) project.

Consistent self-service APIs can be provided across dev, staging, and
production environments, making it easy for app teams to get the infrastructure
they need using vetted infrastructure configurations that meet the standards
of your organization.

## Build Your Own Internal Cloud Platform

App teams can provision the infrastructure they need with a single YAML file
alongside `Deployments` and `Services` using existing tools and workflows
including tools like `kubectl` and Flux to consume your platform's self-service
APIs.

The Platform `Configuration` defines the self-service APIs and
classes-of-service for each API:

* `CompositeResourceDefinitions` (XRDs) define the platform's self-service
   APIs - e.g. `CompositePostgreSQLInstance`.
* `Compositions` offer the classes-of-service supported for each self-service
   API - e.g. `Standard`, `Performance`, `Replicated`.

![Upbound Overview](docs/media/compose.png)

Crossplane `Providers` include the cloud service primitives (AWS, Azure, GCP,
Alibaba) used in a `Composition`.

Learn more about `Composition` in the [Crossplane
Docs](https://crossplane.io/docs/v1.6/concepts/composition.html).

## Quick Start

### Platform Ops/SRE: Run your own internal cloud platform

There are two ways to run Universal Crossplane:

1. Hosted on Upbound Cloud
1. Self-hosted on any Kubernetes cluster.

To provision the Azure Reference platform, you can pick the option that is best for you. 

We'll go through each option in the next sections.

### Upbound Cloud Hosted UXP Control Plane

Hosted Control planes are run on Upbound's cloud infrastructure and provide a restricted
Kubernetes API endpoint that can be accessed via `kubectl` or CI/CD systems.

#### Create a free account in Upbound Cloud

1. Sign up for [Upbound Cloud](https://cloud.upbound.io/register).
1. When you first create an Upbound Account, you can create an Organization

#### Create a Hosted UXP Control Plane in Upbound Cloud

1. Create a `Control Plane` in Upbound Cloud (e.g. dev, staging, or prod).
1. Connect `kubectl` to your `Control Plane` instance.
   * Click on your Control Plane
   * Select the *Connect Using CLI*
   * Paste the commands to configure your local `kubectl` context
   * Test your connectivity by running `kubectl get pods -n upbound-system`

#### Installing UXP on a Kubernetes Cluster

The other option is installing UXP into a Kubernetes cluster you manage using `up`, which
is the official CLI for interacting with Upbound Cloud and Universal Crossplane (UXP).

There are multiple ways to [install up](https://cloud.upbound.io/docs/cli/#install-script),
including Homebrew and Linux packages.

```console
curl -sL https://cli.upbound.io | sh
```

Ensure that your kubectl context is pointing to the correct cluster:

```console
kubectl config current-context
```

Install UXP into the `upbound-system` namespace:

```console
up uxp install
```

Validate the install using the following command:

```console
kubectl get all -n upbound-system
```

#### Install the Crossplane kubectl extension (for convenience)

Now that your kubectl context is configured to connect to a UXP Control Plane,
we can install this reference platform as a Crossplane package.

```console
curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh
cp kubectl-crossplane /usr/local/bin
```

#### Install the Platform Configuration

```console
# Check the latest version available in https://cloud.upbound.io/registry/upbound/platform-ref-azure
PLATFORM_VERSION=v0.1.0
PLATFORM_CONFIG=registry.upbound.io/upbound/platform-ref-azure:${PLATFORM_VERSION}

kubectl crossplane install configuration ${PLATFORM_CONFIG}
kubectl get pkg
```

#### Configure Providers in your Platform

A `ProviderConfig` is used to configure Cloud Provider API credentials. Multiple
`ProviderConfig`s can be created, each one pointing to a different credential.

In order to manage resources in Azure, you must provide credentials for an Azure service principal that Crossplane can use to authenticate. This assumes that you have already set up the Azure CLI client with your credentials.

Create a JSON file that contains all the information needed to connect and authenticate to Azure:

```console
# Create service principal with Owner role
az ad sp create-for-rbac --sdk-auth --role Owner --name platform-ref-azure > crossplane-azure-provider-key.json
```

Take note of the `clientID` value from the JSON file that we just created, and save it to an environment variable:

```console
export AZURE_CLIENT_ID=<clientId value from json file>
```

Now add and grant the required permissions to the service principal that will allow it to manage the necessary resources in Azure:

```console
# add required Azure Active Directory permissions
az ad app permission add --id ${AZURE_CLIENT_ID} --api 00000002-0000-0000-c000-000000000000 --api-permissions 1cda74f2-2616-4834-b122-5cb1b07f8a59=Role 78c8a3c8-a07e-4b9e-af1b-b5ccab50a175=Role

# grant (activate) the permissions
az ad app permission grant --id ${AZURE_CLIENT_ID} --api 00000002-0000-0000-c000-000000000000 --expires never
```

You might see an error similar to the following, but that is OK, the permissions should have gone through still:

_Operation failed with status: 'Conflict'. Details: 409 Client Error: Conflict for url: https://graph.windows.net/e7985bc4-a3b3-4f37-b9d2-fa256023b1ae/oauth2PermissionGrants?api-version=1.6_

Finally, you need to grant admin permissions on the Azure Active Directory to the service principal because it will need to create other service principals for your AKSCluster:

```console
# grant admin consent to the service princinpal you created
az ad app permission admin-consent --id "${AZURE_CLIENT_ID}"
```

_Note: You might need Global Administrator role to Grant admin consent for Default Directory. Please contact the administrator of your Azure subscription. To check your role, go to Azure Active Directory -> Roles and administrators. You can find your role(s) by clicking on Your Role (Preview)_

After these steps are completed, you should have the following file on your local filesystem:

- crossplane-azure-provider-key.json

#### Setup Azure ProviderConfig

Before creating any resources, we need to create and configure an Azure cloud provider resource in Crossplane, which stores the cloud account information in it. All the requests from Crossplane to Azure Cloud will use the credentials attached to this provider resource. The following command assumes that you have a crossplane-azure-provider-key.json file that belongs to the account you’d like Crossplane to use.

Now we’ll create our Secret that contains the credential and ProviderConfig resource that refers to that secret:

```console
kubectl create secret generic azure-account-creds -n upbound-system --from-file=credentials=./crossplane-azure-provider-key.json
kubectl apply -f examples/azure-default-provider.yaml
```

The output will look like the following:

```shell
provider.azure.crossplane.io/default created
```

Crossplane resources use the ProviderConfig named ```default``` if no specific ProviderConfig is specified, so this ProviderConfig will be the default for all Azure resources.

### We are now ready to provision resources:

#### Create Network Fabric

The example network composition includes the creation of a Resource Group, Virtual Network, and Subnets:

```console
kubectl apply -f examples/network.yaml
```

Verify status:

```console
kubectl get claim
kubectl get composite
kubectl get managed
```

#### Create AKS Cluster

```console
kubectl apply -f examples/cluster.yaml
```

verify status:

```console
kubectl get claim
kubectl get composite
kubectl get managed
```

#### Invite App Teams to you Organization in Upbound Cloud

1. Create a Team `team1`.
1. Invite app team members and grant access to `Control Planes` and `Repositories`.

App Dev/Ops: Consume the infrastructure you need using kubectl

#### Join your Organization in Upbound Cloud

1. **Join** your [Upbound Cloud](https://cloud.upbound.io/register)
   `Organization`
1. Verify access to your team `Control Planes` and Registries

#### Provision a PostgreSQLInstance using kubectl

```console
kubectl apply -f examples/postgres-claim.yaml
```

Verify status:

```console
kubectl get claim
kubectl get composite
kubectl get managed
```

### Cleanup & Uninstall

#### Cleanup Resources

Delete resources created through the `Control Plane` Configurations menu:

```console
kubectl delete -f examples/postgres-claim.yaml
kubectl delete -f examples/cluster.yaml
kubectl delete -f examples/network.yaml
```

Verify all underlying resources have been cleanly deleted:

```console
kubectl get managed
```

#### Uninstall Provider & Platform Configuration

```console
kubectl delete configurations.pkg.crossplane.io platform-ref-azure
kubectl delete providers.pkg.crossplane.io provider-azure
kubectl delete providers.pkg.crossplane.io provider-helm
```

### Uninstall Azure App Registration

```console
AZ_APP_ID=$(az ad sp list --display-name platform-ref-azure)
az ad sp delete --id $AZ_APP_ID
```

## APIs in this Configuration

* `Cluster` - provision a fully configured AKS cluster
  * [definition.yaml](cluster/definition.yaml)
  * [composition.yaml](cluster/composition.yaml) includes (transitively):
    * `AKSCluster`
    * `HelmReleases` for Prometheus and other cluster services.
* `Network` - fabric for a `Cluster` to securely connect to Data Services and
  the Internet.
  * [definition.yaml](network/definition.yaml)
  * [composition.yaml](network/composition.yaml) includes:
    * `ResourceGroup`
    * `VirtualNetwork`
    * `Subnet`
* `PostgreSQLInstance` - provision an Azure Database for PostgreSQL instance that securely connects to a `Cluster`
  * [definition.yaml](database/postgres/definition.yaml)
  * [composition.yaml](database/postgres/composition.yaml) includes:
    * `PostgreSQLServer`
    * `PostgreSQLServerVirtualNetworkRule`

## Customize for your Organization

Create a `Repository` called `platform-ref-azure` in your Upbound Cloud `Organization`:

![Upbound Repository](docs/media/repository.png)

Set these to match your settings:

```console
UPBOUND_ORG=acme
UPBOUND_ACCOUNT_EMAIL=me@acme.com
REPO=platform-ref-azure
VERSION_TAG=v0.1.0
REGISTRY=registry.upbound.io
PLATFORM_CONFIG=${REGISTRY:+$REGISTRY/}${UPBOUND_ORG}/${REPO}:${VERSION_TAG}
```

Clone the GitHub repo.

```console
git clone https://github.com/upbound/platform-ref-azure.git
cd platform-ref-azure
```

Login to your container registry.

```console
docker login ${REGISTRY} -u ${UPBOUND_ACCOUNT_EMAIL}
```

Build package.

```console
up xpkg build --name platform-ref-azure.xpkg --ignore ".github/workflows/*,examples/*,hack/*" 
```

Push package to registry.

```console
up xpkg push ${PLATFORM_CONFIG} -f platform-ref-azure.xpkg
```

Install package into an Upbound `Control Plane` instance.

```console
kubectl crossplane install configuration ${PLATFORM_CONFIG}
```

The Azure cloud service primitives that can be used in a `Composition` today are
listed in the [Crossplane Azure Provider
Docs](https://doc.crds.dev/github.com/crossplane/provider-azure).

To learn more see [Configuration
Packages](https://crossplane.io/docs/v0.13/getting-started/package-infrastructure.html).

## What's Next

If you're interested in building your own reference platform for your company,
we'd love to hear from you and chat. You can setup some time with us at
info@upbound.io.

For Crossplane questions, drop by [slack.crossplane.io](https://slack.crossplane.io), and say hi!
