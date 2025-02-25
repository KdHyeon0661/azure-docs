---
title: Expose an AKS service over HTTP or HTTPS using Application Gateway
description: This article provides information on how to expose an AKS service over HTTP or HTTPS using Application Gateway.
services: application-gateway
author: greg-lindsay
ms.service: application-gateway
ms.custom: linux-related-content
ms.topic: how-to
ms.date: 07/23/2023
ms.author: greglin
---

# Expose an AKS service over HTTP or HTTPS using Application Gateway

These tutorials help illustrate the usage of [Kubernetes Ingress Resources](https://kubernetes.io/docs/concepts/services-networking/ingress/) to expose an example Kubernetes service through the [Azure Application Gateway](https://azure.microsoft.com/services/application-gateway/) over HTTP or HTTPS.

> [!TIP]
> Also see [What is Application Gateway for Containers?](for-containers/overview.md) currently in public preview.

## Prerequisites

- Installed `ingress-azure` helm chart.
  - [**Greenfield Deployment**](ingress-controller-install-new.md): If you're starting from scratch, refer to these installation instructions, which outlines steps to deploy an AKS cluster with Application Gateway and install application gateway ingress controller on the AKS cluster.
  - [**Brownfield Deployment**](ingress-controller-install-existing.md): If you have an existing AKS cluster and Application Gateway, refer to these instructions to install application gateway ingress controller on the AKS cluster.
- If you want to use HTTPS on this application, you need an x509 certificate and its private key.

## Deploy `guestbook` application

The guestbook application is a canonical Kubernetes application that composes of a Web UI frontend, a backend and a Redis database. By default, `guestbook` exposes its application through a service with name `frontend` on port `80`. Without a Kubernetes Ingress Resource, the service isn't accessible from outside the AKS cluster. We use the application and set up Ingress Resources to access the application through HTTP and HTTPS.

Use the following instructions to deploy the guestbook application.

1. Download `guestbook-all-in-one.yaml` from [here](https://raw.githubusercontent.com/kubernetes/examples/master/guestbook/all-in-one/guestbook-all-in-one.yaml)
1. Deploy `guestbook-all-in-one.yaml` into your AKS cluster by running

  ```bash
  kubectl apply -f guestbook-all-in-one.yaml
  ```

Now, the `guestbook` application has been deployed.

## Expose services over HTTP

To expose the guestbook application, use the following ingress resource:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: guestbook
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
spec:
  rules:
  - http:
      paths:
      - backend:
          serviceName: frontend
          servicePort: 80
```

This ingress exposes the `frontend` service of the `guestbook-all-in-one` deployment
as a default backend of the Application Gateway.

Save the above ingress resource as `ing-guestbook.yaml`.

1. Deploy `ing-guestbook.yaml` by running:

    ```bash
    kubectl apply -f ing-guestbook.yaml
    ```

1. Check the log of the ingress controller for deployment status.

Now the `guestbook` application should be available. You can check availability by visiting the public address of the Application Gateway.

## Expose services over HTTPS

### Without specified hostname

Without specifying hostname, the guestbook service is available on all the host-names pointing to the application gateway.

1. Before deploying ingress, you need to create a kubernetes secret to host the certificate and private key. You can create a kubernetes secret by running

    ```bash
    kubectl create secret tls <guestbook-secret-name> --key <path-to-key> --cert <path-to-cert>
    ```

1. Define the following ingress. In the ingress, specify the name of the secret in the `secretName` section.

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: guestbook
      annotations:
        kubernetes.io/ingress.class: azure/application-gateway
    spec:
      tls:
        - secretName: <guestbook-secret-name>
      rules:
      - http:
          paths:
          - backend:
              serviceName: frontend
              servicePort: 80
    ```

    > [!NOTE]
    > Replace `<guestbook-secret-name>` in the above Ingress Resource with the name of your secret. Store the above Ingress Resource in a file name `ing-guestbook-tls.yaml`.

1. Deploy ing-guestbook-tls.yaml by running

    ```bash
    kubectl apply -f ing-guestbook-tls.yaml
    ```

1. Check the log of the ingress controller for deployment status.

Now the `guestbook` application is available on both HTTP and HTTPS.

### With specified hostname

You can also specify the hostname on the ingress in order to multiplex TLS configurations and services.
By specifying hostname, the guestbook service is only available on the specified host.

1. Define the following ingress.
    In the ingress, specify the name of the secret in the `secretName` section and replace the hostname in the `hosts` section accordingly.

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: guestbook
      annotations:
        kubernetes.io/ingress.class: azure/application-gateway
    spec:
      tls:
        - hosts:
          - <guestbook.contoso.com>
          secretName: <guestbook-secret-name>
      rules:
      - host: <guestbook.contoso.com>
        http:
          paths:
          - backend:
              serviceName: frontend
              servicePort: 80
    ```

1. Deploy `ing-guestbook-tls-sni.yaml` by running

    ```bash
    kubectl apply -f ing-guestbook-tls-sni.yaml
    ```

1. Check the log of the ingress controller for deployment status.

Now the `guestbook` application is available on both HTTP and HTTPS only on the specified host (`<guestbook.contoso.com>` in this example).

## Integrate with other services

The following ingress allows you to add other paths into this ingress and redirect those paths to other services:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: guestbook
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
spec:
  rules:
  - http:
      paths:
      - path: </other/*>
        backend:
          serviceName: <other-service>
          servicePort: 80
       - backend:
          serviceName: frontend
          servicePort: 80
```
