# Snyk Local Code Engine <!-- omit in toc -->

- [Getting Started](#getting-started)
- [Installation](#installation)
  - [Enabling CLI and/or PR Checks](#enabling-cli-andor-pr-checks)
  - [Cluster with Cloud Load Balancer](#cluster-with-cloud-load-balancer)
    - [Static IP](#static-ip)
      - [GCP](#gcp)
      - [AWS](#aws)
    - [Internal Load Balancer Additional Configuration](#internal-load-balancer-additional-configuration)
      - [GCP](#gcp-1)
      - [AWS](#aws-1)
    - [Configure Local Engine URL](#configure-local-engine-url)
    - [Checking Cluster Health](#checking-cluster-health)
    - [Next Steps](#next-steps)
  - [Cluster without Cloud Load Balancer/ Self Hosted Cluster](#cluster-without-cloud-load-balancer-self-hosted-cluster)
    - [Configure Local Engine URL - No Load Balancer/Self Hosted](#configure-local-engine-url---no-load-balancerself-hosted)
  - [Enabling TLS](#enabling-tls)
    - [Adding a Certificate](#adding-a-certificate)
    - [Self Signed Certificates](#self-signed-certificates)
  - [Parameters](#parameters)
    - [Global required parameters](#global-required-parameters)
    - [Global optional parameters](#global-optional-parameters)
    - [Broker parameters](#broker-parameters)
      - [GitHub.com parameters](#githubcom-parameters)
      - [GitHub Enterprise parameters](#github-enterprise-parameters)
      - [GitLab parameters](#gitlab-parameters)
      - [Bitbucket server parameters](#bitbucket-server-parameters)
      - [Azure Repos parameters](#azure-repos-parameters)
    - [broker-client](#broker-client)
    - [codeapi](#codeapi)
    - [bundle](#bundle)
    - [suggest](#suggest)
    - [deeproxy](#deeproxy)
    - [nginx-ingress-controller](#nginx-ingress-controller)
    - [service-health-aggregator](#service-health-aggregator)
- [Third Party charts](#third-party-charts)
- [Logging](#logging)
- [Uninstalling Snyk Code Local Engine](#uninstalling-snyk-code-local-engine)

## Introduction

This documentation details how Snyk Local Code Engine can be deployed on a Kubernetes cluster using Helm package manager.

## Prerequisites

- Kubernetes Cluster running version 1.16.0 - 1.21.5
- 3 nodes with the following specs:
  - CPU 32 cores
  - RAM 120GB
  - Disk 500 GB (>300GB Ephemeral Storage)
- Helm 3.8.0

# Getting Started

Snyk Code Local Engine is installed and configured with Helm.

Image Pull Secrets and Broker Settings are required for basic installation. We recommend that settings are configured in `values-customer-settings.yaml` to simplify upgrades and troubleshooting, making installation/updates as simple as:

```bash
helm upgrade <INSTANCE_NAME> code-<VERSION>.tgz -i -f values-customer-settings.yaml
```

`INSTANCE_NAME` can be a company name or another name meaningful to you. `VERSION` should match the Helm Chart `.tgz` archive. Make sure that you get correct path to the `values-customer-settings.yaml` configuration file - Snyk Code Local Engine cannot be installed without it.

If you would rather not include image pull secrets in `values-customer-settings.yaml`, they can be included as arguments when running the Helm chart:

```bash
helm upgrade <INSTANCE_NAME> code-<VERSION>.tgz -i -f values-customer-settings.yaml \
--set global.imagePullSecret.credentials.username="<SNYK_REGISTRY_USER>" \
--set global.imagePullSecret.credentials.password='<SNYK_REGISTRY_PASSWORD>'
```

The above commands installs (or if a release already exists, upgrades) Snyk Code Local Engine. If the deployment was successful you will be able to scan a project via the Snyk Web Portal using whichever SCM integration specified.

# Installation

## Enabling CLI and/or PR Checks

Snyk Code Local Engine services needs to be accessible from outside the cluster for CLI and/or PR checks to work. Deployment settings will vary depending on how the cluster is configured. In all cases, cluster-specific settings are an addition to the settings in [Getting Started](#getting-started).

Add the following to `values-customer-settings.yaml`:

```yaml
global:
  ingress:
    enabled: true
```

Further steps to enable CLI and/or PR Checks vary depending on your cluster - skip to the section that is relevant to you:

- [Cluster with Cloud Load Balancer](#cluster-with-cloud-load-balancer)
- [Cluster without Cloud Load Balancer / Self Hosted Cluster](#cluster-without-cloud-load-balancer)

## Cluster with Cloud Load Balancer

_Before reading this section, ensure Snyk Code Local Engine has been installed on your cluster._

Snyk Code Local Engine supports Load Balancers from AWS and GCP. The Load Balancer may be `external` (the default) or `internal` (extra configuration required).

If your cluster is deployed on AWS or GCP, make changes in `values-customer-settings.yaml` as described in [Enabling CLI and/or PR Checks](#enabling-cli-andor-pr-checks) and upgrade the release via `helm upgrade`.

This will instruct your Cloud Provider to provision a new Load Balancer for your cluster - by default an external Load Balancer is created and is allocated a dynamic IP address by your Cloud Provider. If an internal Load Balancer is needed (no access from the internet) follow the [Internal Load Balancer](#internal-load-balancer-additional-configuration) section.

The IP address for any DNS records is the `EXTERNAL-IP` of the `nginx-ingress-controller` `Service` which may be acquired via:

```bash
$ kubectl get svc <INSTANCE_NAME>-nginx-ingress-controller

> NAME                                        TYPE           CLUSTER-IP   EXTERNAL-IP   ...
  <INSTANCE_NAME>-nginx-ingress-controller    LoadBalancer   x.x.x.x      x.x.x.x       ...
```

As this dynamic IP may change if released, if Snyk Code Local Engine is uninstalled and redeployed you _may_ have to update any DNS records that resolve to this dynamic IP. See the following section to [set a Static IP](#static-ip) for your cluster if required, or jump straight to [Configuring Local Engine URL](#configure-local-engine-url).

### Static IP

If you have a static IP from your Cloud provider, this may be added as follows:

#### GCP

- If using an `external` Load Balancer, reserve a new [external](https://cloud.google.com/compute/docs/ip-addresses/reserve-static-external-ip-address#reserve_new_static) IP address. If an `internal` Load Balancer is required, reserve an [internal](https://cloud.google.com/compute/docs/ip-addresses/reserve-static-internal-ip-address) IP address in the _same GCP zone as your cluster_, and follow [Internal Load Balancer Additional Configuration](#gcp-1) steps.

- Add the following to `values-customer-settings.yaml`, replacing `STATIC_IP_ADDRESS_HERE` with the **Public** IP Address:

```yaml
nginx-ingress-controller:
  service:
    loadBalancerIP: <STATIC_IP_ADDRESS_HERE>
```

- Execute ` helm upgrade <INSTANCE_NAME> code-<VERSION>.tgz -i -f values-customer-settings.yaml`

- [Configure Local Engine URL](#configure-local-engine-url) with your new static IP address.

#### AWS

- Allocating a static IP address requires the use of an external Network Load Balancer - this is not provisioned/managed within Snyk Code Local Engine. Follow the steps described in this [Amazon Support Topic](https://aws.amazon.com/premiumsupport/knowledge-center/alb-static-ip/).

- [Configure Local Engine URL](#configure-local-engine-url) to point to the static IP address of the _Network Load Balancer_ once provisioned.

- If Snyk Code Local Engine is uninstalled and redployed the Network Load Balancer mapping may need to be re-created.

### Internal Load Balancer Additional Configuration

If an internal LoadBalancer is required annotations must be added to `values-customer-settings.yaml`. This is dependant on your Cloud provider. Either an internal DNS record or a static IP is recommended for this deployment model.

Snippets are provided, and are case-sensitive:

#### GCP

- Add the following to `values-customer-settings.yaml`:

```yaml
nginx-ingress-controller:
  service:
    annotations:
      networking.gke.io/load-balancer-type: "Internal"
```

- Execute ` helm upgrade <INSTANCE_NAME> code-<VERSION>.tgz -i -f values-customer-settings.yaml`

This resource will be allocated a dyanmic `internal` IP address and will not be reachable from the internet. If a static IP is also required, follow [Static IP instructions for GCP](#gcp).

#### AWS

- Add the following to `values-customer-settings.yaml`:

```yaml
nginx-ingress-controller:
  service:
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-scheme: "internal"
```

- Execute ` helm upgrade <INSTANCE_NAME> code-<VERSION>.tgz -i -f values-customer-settings.yaml`

This resource will be allocated a dyanmic `internal` IP address and will not be reachable from the internet. If a static IP is also required, follow [Static IP instructions for AWS](#aws).

### Configure Local Engine URL

_Note: requires Snyk Code Local Engine be installed on your cluster._

It is recommended to create a DNS record (either internal or external, depending on your deployment) to simplify the access of your installation. The IP address may be used directly if a DNS record is not created.

Add the following to `values-customer-settings.yaml` and upgrade Snyk Code Local Engine:

```yaml
global:
  ...
  localEngineUrl: <either your assigned DNS record or your IP address - for example, http://local-engine.domain or http:/10.170.1.40>
```

**If an IP address was previously provided to your Snyk representative, please update us with your new DNS record.**

### Checking Cluster Health

To check if everything works, run:

```bash
curl -L http://<LOCAL_ENGINE_URL>/
> 302 -> <LOCAL_ENGINE_URL>/status
```

and

```bash
curl http://<LOCAL_ENGINE_URL>/broker/healthcheck
```

and

```bash
curl http://<LOCAL_ENGINE_URL>/api/healthcheck
```

For the first check, the response should be: `"Snyk Code is healthy! 🐶"` - if not, insepct the data that is returned. Some pods may not be in a healthy state.

For the second, you should see something similar: `{"ok":true,"websocketConnectionOpen":true,"brokerServerUrl":"https://broker.snyk.io","version":"4.116.0"}`

For the third, you should see: `{"gitSha":"sha","ok":true}`

### Next Steps

After configuring and installing Snyk Local Code Engine, please **contact your Snyk representative with your Snyk Code Local Engine domain or static IP address so this may be configured on Snyk’s systems.**

---

## Cluster without Cloud Load Balancer/ Self Hosted Cluster

_Note: This section does not apply for clusters using Cloud Native Load Balancers. See [Cluster with Cloud Load Balancer](#cluster-with-cloud-load-balancer) for those instructions._

1. Make changes in `values-customer-settings.yaml` as described in [Enabling CLI and/or PR Checks](#enabling-cli-andor-pr-checks) if required.

2. Make the following changes to `values-customer-settings.yaml` and run `helm upgrade`:

```yaml
nginx-ingress-controller:
  service:
    nodePorts:
      http: <PORT>
    type: "NodePort"
```

Where `<PORT>` is any port that falls into Kubenetes NodePort's range (`30000` - `32767`). We recommend to restrict access to this IP address to only known safe sources.

1. Identify the `NODE_IP` of the `nginx-ingress-controller`'s pod. You can get it using:

```bash
$ kubectl get pods -l app.kubernetes.io/name=nginx-ingress-controller -l app.kubernetes.io/component=controller -o wide
```

_Note: Whilst any Node IP will allow a connection to Snyk Code Local Engine, this allows requests to go directly to NGINX without any additional node-to-node forwarding by Kubernetes._

1. Check if this setup was successful by running the [healthcheck steps](#checking-cluster-health). Your `IP/DNS` will be `<NODE_IP>:<PORT>`.

1. Contact your Snyk representative to configure your ingress IP and port on Snyk’s systems.

### Configure Local Engine URL - No Load Balancer/Self Hosted

_Note: This section does not apply for customers using Cloud Native Load Balancers. See [Configure Local Engine URL](#configure-local-engine-url) for those instructions._

If your nodes have static IP addresses, create an `A` record per node. Any requests to this domain will be received by the `NodePort` Kubernetes service, which routes the request to whichever pod runs `nginx-ingress-controller`.

If your nodes have dynamic IP addresses creating DNS records may be operationally complex - it is recommended to seek support within your organisation.

---

## Enabling TLS

_Note: A DNS record is required to enable TLS. See [Configure Local Engine URL](#configure-local-engine-url) or [Configure Local Engine URL - No Load Balancer/Self Hosted](#configure-local-engine-url---no-load-balancerself-hosted) for instructions._

After making any changes described in this section **please contact your Snyk representative.**

### Adding a Certificate

By default, the ingress endpoints are not secured by TLS. In order to secure them:
- a DNS record must be created,
- a certificate generated,
- the following changes made in the `values-customer-settings.yaml` file:

```yaml
global:
  ...
  ingress:
    ...
    host: "<HOST>"
    tls:
      enabled: true
      secret:
        ...
        key: |
          <KEY>
        cert: |
          <CERT>
```

Where `<HOST>` is the domain used when the TLS `<CERT>` and `<KEY>` were generated. The Helm value `global.localEngineUrl` should match `<HOST>`.

Read more about securing an ingress with TLS [here](https://kubernetes.io/docs/concepts/services-networking/ingress/#tls) and TLS secrets [in general](https://kubernetes.io/docs/concepts/configuration/secret/#tls-secrets).

### Self Signed Certificates

If using a self-signed certificate, append the `--insecure` option to any `snyk` commands, or provide the CA to `snyk` by setting `NODE_EXTRA_CA_CERTS`. This [help article](https://support.snyk.io/hc/en-us/articles/360000925358-How-can-I-use-Snyk-behind-a-proxy-) provides further information and support.

---

## Parameters

### Global required parameters

| Name                                          | Description                  | Default Value |
| --------------------------------------------- | ---------------------------- | ------------- |
| `global.imagePullSecret.credentials.username` | Docker hub registry username | `""`          |
| `global.imagePullSecret.credentials.password` | Docker hub registry password | `""`          |

### Global optional parameters

| Name                                    | Description                                                                        | Default Value                       |
| --------------------------------------- | ---------------------------------------------------------------------------------- | ----------------------------------- |
| `global.ingress.enable`                 | Enable Ingress                                                                     | `false`                             |
| `global.ingress.installController`      | Nginx ingress controller - required if no cloud-based ingress controller is in use | `false`                             |
| `global.ingress.ingressClassName`       | Ingress controller class name                                                      | `"nginx"`                           |
| `global.ingress.host`                   | Ingress host                                                                       | `""`                                |
| `global.ingress.annotations`            | Additional annotations for the ingress resource                                    | `{}`                                |
| `global.ingress.tls.enabled`            | Enable TLS for the ingress endpoints                                               | `false`                             |
| `global.ingress.tls.secret.name`        | Name of the secret to use for the ingress.                                         | `"{{ .Release.Name }}-ingress-tls"` |
| `global.ingress.tls.secret.key`         | TLS key that will be used to create an ingress secret                              | `""`                                |
| `global.ingress.tls.secret.cert`        | TLS certificate that will be used to create an ingress secret                      | `""`                                |
| `global.ingress.tls.secret.annotations` | Additional annotations for the auto-generated TLS secret resource                  | `{}`                                |

### Broker parameters

| Name                       | Description                                                                                         | Default Value |
| -------------------------- | --------------------------------------------------------------------------------------------------- | ------------- |
| `broker-client.brokerType` | Type of broker client to use. Supported options: github-com, github-enterprise, gitlab, azure-repos | `""`          |

Please use _one_ of the following SCM configuration

#### GitHub.com parameters

| Name                        | Description                   | Default Value |
| --------------------------- | ----------------------------- | ------------- |
| `broker-client.githubToken` | Token for <http://github.com> | `""`          |

#### GitHub Enterprise parameters

| Name                          | Description                              | Default Value                                     |
| ----------------------------- | ---------------------------------------- | ------------------------------------------------- |
| `broker-client.githubToken`   | Token for <http://github.com>            | `""`                                              |
| `broker-client.githubHost`    | Host name for GitHub server              | `""`                                              |
| `broker-client.githubApi`     | GitHub REST API url, excluding scheme    | `"{{ .Values.broker-client.githubHost }}/api/v3"` |
| `broker-client.githubGraphql` | GitHub GraphQL API url, excluding scheme | `"{{ .Values.broker-client.githubHost }}/api"`    |

#### GitLab parameters

| Name                        | Description                 | Default Value |
| --------------------------- | --------------------------- | ------------- |
| `broker-client.gitlabToken` | GitLab Host                 | `""`          |
| `broker-client.gitlabHost`  | Host name for gitlab server | `""`          |

#### Bitbucket server parameters

| Name                              | Description                    | Default Value |
| --------------------------------- | ------------------------------ | ------------- |
| `broker-client.bitbucketUsername` | Bitbucket username             | `""`          |
| `broker-client.bitbucketPassword` | Bitbucket password             | `""`          |
| `broker-client.bitbucketHost`     | Host name for bitbucket server | `""`          |

#### Azure Repos parameters

| Name                            | Description               | Default Value |
| ------------------------------- | ------------------------- | ------------- |
| `broker-client.azureReposToken` | Token for Azure Repos     | `""`          |
| `broker-client.azureReposHost`  | Host name for Azure Repos | `""`          |
| `broker-client.azureReposOrg`   | Azure organization name   | `""`          |

### broker-client

| Name                                           | Description                                                                                     | Default Value |
| ---------------------------------------------- | ----------------------------------------------------------------------------------------------- | ------------- |
| `broker-client.codeSnippet.enabled`            | Enable code Snippets                                                                            | `false`       |
| `broker-client.imagePullSecrets`               | Docker registry secret names as an array                                                        | `[]`          |
| `broker-client.nameOverride`                   | String to partially override names.fullname template (will maintain the release name)           | `""`          |
| `broker-client.fullnameOverride`               | String to fully override names.fullname template                                                | `""`          |
| `broker-client.serviceAccount.create`          | Specify whether a ServiceAccount should be created                                              | `true`        |
| `broker-client.serviceaccount.name`            | The name of the ServiceAccount to create                                                        | `""`          |
| `broker-client.serviceAccount.annotations`     | Additional Service Account annotations (evaluated as a template)                                | `{}`          |
| `broker-client.podAnnotations`                 | Pod annotations                                                                                 | `{}`          |
| `broker-client.podSecurityContext`             | Security context for pod                                                                        | `{}`          |
| `broker-client.securityContext`                | Holds security configuration that will be applied to a container                                | `{}`          |
| `broker-client.nodeSelector`                   | Aggregator Node labels for pod assignment                                                       | `{}`          |
| `broker-client.tolerations`                    | Aggregator Tolerations for pod assignment                                                       | `[]`          |
| `broker-client.affinity`                       | Forwarder Affinity for pod assignment                                                           | `{}`          |
| `broker-client.ingress.enabled`                | Enable exposing the client via an ingress                                                       | `false`       |
| `broker-client.ingress.host`                   | Specifies which host the ingress will be configured for                                         | `""`          |
| `broker-client.ingress.annotations`            | Additional annotations for the ingress resource                                                 | `{}`          |
| `broker-client.ingress.tls.enabled`            | Enable TLS for the ingress endpoints                                                            | `false`       |
| `broker-client.ingress.tls.secret.name`        | Name of the secret to use for the ingress. Leave empty to default to `{servicename}-tls-secret` | `""`          |
| `broker-client.ingress.tls.secret.key`         | TLS key that will be used to create an ingress secret                                           | `""`          |
| `broker-client.ingress.tls.secret.cert`        | TLS certificate that will be used to create an ingress secret                                   | `""`          |
| `broker-client.ingress.tls.secret.annotations` | Additional annotations for the auto-generated TLS secret resource                               | `{}`          |

### codeapi

| Name                                 | Description                                                                           | Default Value |
| ------------------------------------ | ------------------------------------------------------------------------------------- | ------------- |
| `codeapi.imagePullSecrets`           | Docker registry secret names as an array                                              | `[]`          |
| `codeapi.nameOverride`               | String to partially override names.fullname template (will maintain the release name) | `""`          |
| `codeapi.fullnameOverride`           | String to fully override names.fullname template                                      | `""`          |
| `codeapi.serviceAccount.create`      | Specify whether a ServiceAccount should be created                                    | `true`        |
| `codeapi.serviceaccount.name`        | The name of the ServiceAccount to create                                              | `""`          |
| `codeapi.serviceAccount.annotations` | Additional Service Account annotations (evaluated as a template)                      | `{}`          |
| `codeapi.podAnnotations`             | Pod annotations                                                                       | `{}`          |
| `codeapi.podSecurityContext`         | Security context for pod                                                              | `{}`          |
| `codeapi.securityContext`            | holds security configuration that will be applied to a container                      | `{}`          |
| `codeapi.nodeSelector`               | Aggregator Node labels for pod assignment                                             | `{}`          |
| `codeapi.tolerations`                | Aggregator Tolerations for pod assignment                                             | `[]`          |
| `codeapi.affinity`                   | Forwarder Affinity for pod assignment                                                 | `{}`          |

### bundle

| Name                                | Description                                                                           | Default Value |
| ----------------------------------- | ------------------------------------------------------------------------------------- | ------------- |
| `bundle.imagePullSecrets`           | Docker registry secret names as an array                                              | `[]`          |
| `bundle.nameOverride`               | String to partially override names.fullname template (will maintain the release name) | `""`          |
| `bundle.fullnameOverride`           | String to fully override names.fullname template                                      | `""`          |
| `bundle.serviceAccount.create`      | Specify whether a ServiceAccount should be created                                    | `true`        |
| `bundle.serviceaccount.name`        | The name of the ServiceAccount to create                                              | `""`          |
| `bundle.serviceAccount.annotations` | Additional Service Account annotations (evaluated as a template)                      | `{}`          |
| `bundle.podAnnotations`             | Pod annotations                                                                       | `{}`          |
| `bundle.podSecurityContext`         | Security context for pod                                                              | `{}`          |
| `bundle.securityContext`            | Holds security configuration that will be applied to a container                      | `{}`          |
| `bundle.nodeSelector`               | Aggregator Node labels for pod assignment                                             | `{}`          |
| `bundle.tolerations`                | Aggregator Tolerations for pod assignment                                             | `[]`          |
| `bundle.affinity`                   | Forwarder Affinity for pod assignment                                                 | `{}`          |

### suggest

| Name                                 | Description                                                                           | Default Value |
| ------------------------------------ | ------------------------------------------------------------------------------------- | ------------- |
| `suggest.imagePullSecrets`           | Docker registry secret names as an array                                              | `[]`          |
| `suggest.nameOverride`               | String to partially override names.fullname template (will maintain the release name) | `""`          |
| `suggest.fullnameOverride`           | String to fully override names.fullname template                                      | `""`          |
| `suggest.serviceAccount.create`      | Specify whether a ServiceAccount should be created                                    | `true`        |
| `suggest.serviceaccount.name`        | The name of the ServiceAccount to create                                              | `""`          |
| `suggest.serviceAccount.annotations` | Additional Service Account annotations (evaluated as a template)                      | `{}`          |
| `suggest.podAnnotations`             | Pod annotations                                                                       | `{}`          |
| `suggest.podSecurityContext`         | Security context for pod                                                              | `{}`          |
| `suggest.securityContext`            | holds security configuration that will be applied to a container                      | `{}`          |
| `suggest.nodeSelector`               | Aggregator Node labels for pod assignment                                             | `{}`          |
| `suggest.tolerations`                | Aggregator Tolerations for pod assignment                                             | `[]`          |
| `suggest.affinity`                   | Forwarder Affinity for pod assignment                                                 | `{}`          |

### deeproxy

| Name                                  | Description                                                      | Default Value |
| ------------------------------------- | ---------------------------------------------------------------- | ------------- |
| `deeproxy.serviceAccount.create`      | Specify whether a ServiceAccount should be created               | `true`        |
| `deeproxy.serviceaccount.name`        | The name of the ServiceAccount to create                         | `""`          |
| `deeproxy.serviceAccount.annotations` | Additional Service Account annotations (evaluated as a template) | `{}`          |
| `deeproxy.nodeSelector`               | Aggregator Node labels for pod assignment                        | `{}`          |
| `deeproxy.tolerations`                | Aggregator Tolerations for pod assignment                        | `[]`          |
| `deeproxy.affinity`                   | Forwarder Affinity for pod assignment                            | `{}`          |

### nginx-ingress-controller

| Name                                               | Description                      | Default Value    |
| -------------------------------------------------- | -------------------------------- | ---------------- |
| `nginx-ingress-controller.service.nodePorts.http`  | Specify the nodePort http value  | `""`             |
| `nginx-ingress-controller.service.nodePorts.https` | Specify the nodePort https value | `""`             |
| `nginx-ingress-controller.service.type`            | Service type                     | `"LoadBalancer"` |

### service-health-aggregator

_Either_ set `clusterRole` or `Role` to be created, do not set both.

| Name                                                           | Description                                                                                                                                                                       | Default Value                      |
| -------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------- |
| `service-health-aggregator.healthchecks`                       | A list of objects with keys `namespace` (`string`), `enabled` (`bool`) and `services` (`[]string`). Leave empty to default to all services (`["*"]`) in the deployment namespace. | `[]`                               |
| `service-health-aggregator.healthyMessage`                     | The message to return when targeted services/pods are healthy                                                                                                                     | `"Targeted Services are healthy."` |
| `service-health-aggregator.imagePullSecrets`                   | Docker registry secret names as an array                                                                                                                                          | `[]`                               |
| `service-health-aggregator.nameOverride`                       | String to partially override names.fullname template (will maintain the release name)                                                                                             | `""`                               |
| `service-health-aggregator.fullnameOverride`                   | String to fully override names.fullname template                                                                                                                                  | `""`                               |
| `service-health-aggregator.role.create`                        | Specify whether a Role should be created                                                                                                                                          | `true`                             |
| `service-health-aggregator.role.annotations`                   | Additional Role annotations (evaluated as a template)                                                                                                                             | `{}`                               |
| `service-health-aggregator.role.permissions.namespaces`        | A list of namespaces to allow access to (leave empty to default to the deployment namespace)                                                                                      | `[]`                               |
| `service-health-aggregator.role.name`                          | The name of a pre-existing Role to use                                                                                                                                            | `""`                               |
| `service-health-aggregator.clusterRole.create`                 | Specify whether a ClusterRole should be created                                                                                                                                   | `true`                             |
| `service-health-aggregator.clusterRole.annotations`            | Additional ClusterRole annotations (evaluated as a template)                                                                                                                      | `{}`                               |
| `service-health-aggregator.clusterRole.permissions.namespaces` | A list of namespaces to allow access to (leave empty to default to the deployment namespace)                                                                                      | `[]`                               |
| `service-health-aggregator.clusterRole.name`                   | The name of a pre-existing ClusterRole to use                                                                                                                                     | `""`                               |
| `service-health-aggregator.serviceAccount.create`              | Specify whether a ServiceAccount should be created                                                                                                                                | `true`                             |
| `service-health-aggregator.serviceAccount.name`                | The name of the ServiceAccount to create                                                                                                                                          | `""`                               |
| `service-health-aggregator.serviceAccount.annotations`         | Additional Service Account annotations (evaluated as a template)                                                                                                                  | `{}`                               |
| `service-health-aggregator.nodeSelector`                       | Aggregator Node labels for pod assignment                                                                                                                                         | `{}`                               |
| `service-health-aggregator.tolerations`                        | Aggregator Tolerations for pod assignment                                                                                                                                         | `[]`                               |
| `service-health-aggregator.affinity`                           | Forwarder Affinity for pod assignment                                                                                                                                             | `{}`                               |

# Third Party charts

To further customise third party services integrated into Snyk Code Local Engine, explore the following examples:

- [Ambassador](https://github.com/emissary-ingress/emissary/tree/master/charts/emissary-ingress) - used for internal load balancing
- [Redis](https://github.com/bitnami/charts/tree/master/bitnami/redis) - used for an in-memory cache
- [Fluentd](https://github.com/bitnami/charts/tree/master/bitnami/fluentd) - used for logging

# Logging

Fluentd is used to aggregate all our services' logs into one file. Logs are written to the `/var/log/snyk/logs` directory and are available on the `fluentd` pod.

The log file is rotated at least once per-day. This file _may_ be rotated multiple times per day if the file exceeds the configured size limit `fluentd-umbrella.logrotate.fileMaxSizeInMb` (default `500`). The number of files `logrotate` keeps is `fluentd-umbrella.logrotate.filesToKeep` (default 14). When a file is rotated the date and time of rotation is added as a suffix to the file name.

Logrotate will validate that the number of log files is not above `fluentd-umbrella.logrotate.filesToKeep` by removing the oldest log file when the number of log files is equal to `filesToKeep`+1 files.

Current logs for Snyk Code Local Engine may be viewed by `tail`ing the `code.log` file. To view log entries in the past, interrogate the log file with date and time _after_ the timeframe you would like to inspect.

To fetch all Snyk Code Local Engine log files to current directory run:

```console
kubectl cp <your-namespace>/<your-release>-fluentd-0:/var/log/snyk/logs ./
```

To fetch log files for specific dates get the list of existing log files by runnning:

```console
kubectl exec -it on-prem-fluentd-0 -- ls -l /var/log/snyk/logs
```

Then copy the log file you want to fetch by running:

```console
kubectl cp <your-namespace>/<your-release>-fluentd-0:/var/log/snyk/logs/<file-name> ./<target-file-name>
```

_NOTE: It may take a few minutes to copy all files - add the `-v` option to `cp` for output when file(s) are copied._

# Uninstalling Snyk Code Local Engine

To uninstall/delete the `INSTANCE_NAME` resources:

```bash
helm delete `INSTANCE_NAME`
```

The command removes all the Kubernetes components associated with the chart and deletes the release from the cluster. The `--purge` option may be used to delete all history if required.
