# SSL on Kubernetes

The quickest way to get automated SSL/TLS certificates for your project if it is expose to the web is to use [cert-manager][cert-manager]. In this tutorial, we will deploy OpenFaaS using the [Helm Chart][openfaas-helm], [cert-manager][cert-manager], and [nginx-ingress][nginx-ingress]

## Create an A record

If your domain is `.domain.com` then create an A record using your DNS administration panel such as `gateway.domain.com` or `openfaas.domain.com`. The required steps will vary depending on your domain provider and your cluster provider. For example; [on Google Cloud DNS](https://cloud.google.com/kubernetes-engine/docs/tutorials/configuring-domain-name-static-ip) or [with Route53 using AWS](https://kubernetes.io/docs/setup/custom-cloud/kops/#2-5-create-a-route53-domain-for-your-cluster).

## Configure Helm and Tiller

First install Helm and the Tiller [following the instructions provided by Helm][helm-install]

## Install OpenFaaS

Follow the instructions found in the [OpenFaaS Helm Chart](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas#deploy-openfaas), make sure to [secure your gateway with basic auth](https://github.com/openfaas/faas-netes/tree/master/chart/openfaas#secure-the-gateway-administrative-api-and-ui-with-basic-auth) before you continue.

## Install nginx-ingress

Add the `nginx-ingress` using

```sh
$ helm install stable/nginx-ingress --name nginxingress --set rbac.create=true
```

The full configuration options for nginx [can be found here][nginx-configuration].

## Install cert-manager

We install it into the `kube-system` namespace using the following command:

```sh
$ helm install \
    --name cert-manager \
    --namespace kube-system \
    stable/cert-manager
```

This configuration will work for most deployments, but you can also see https://cert-manager.readthedocs.io/en/latest/getting-started/2-installing.html for additional instructions and options for installing cert-manager.

## Configure cert-manager

In additional to the controller installed in the previous step, we must also configure an "Issuer" before `cert-manager` can create certificates for our services. For convenience we will create an Issuer for both Let's Encrypt's production API and their staging API. The staging API has much higher rate limits. We will use it to issue a test certificate before switching over to a production certificate if everything works as expected.

Replace `<your-email-here>` with the contact email that will be shown with the SSL certificate.

!!! example "letsencrypt-issuer.yaml"
    ```yaml
    apiVersion: certmanager.k8s.io/v1alpha1
    kind: Issuer
    metadata:
      name: letsencrypt-prod
      namespace: openfaas
    spec:
      acme:
        # Email address used for ACME registration
        email: <your-email-here>
        http01: {}
        # Name of a secret used to store the ACME account private key
        privateKeySecretRef:
          key: ""
          name: letsencrypt-prod
        server: https://acme-v02.api.letsencrypt.org/directory
    ---
    apiVersion: certmanager.k8s.io/v1alpha1
    kind: Issuer
    metadata:
      name: letsencrypt-staging
      namespace: openfaas
    spec:
      acme:
        server: https://acme-staging.api.letsencrypt.org/directory
        # Email address used for ACME registration
        email: <your-email-here>
        # Name of a secret used to store the ACME account private key
        privateKeySecretRef:
          name: letsencrypt-staging
        http01: {}
    ```

```sh
$ kubectl apply -f letsencrypt-issuer.yaml
```

This will allow `cert-manager` to automatically provision Certificates just in the `openfaas` namespace.

## Add TLS to openfaas

The OpenFaaS Helm Chart already supports the nginx-ingress, but we want to customize it further. This is easiest with a custom values file. Below, we enable and configure the ingress object to use our certificate and expose just the gateway

!!! example "tls.yml"
    ```yaml
    ingress:
        enabled: true
        annotations:
            kubernetes.io/ingress.class: nginx
            certmanager.k8s.io/cluster-issuer: letsencrypt-staging
        ingress:
            tls:
            - hosts:
              - openfaas.mydomain.com
              secretName: openfaas-crt
            hosts:
            - host: openfaas.mydomain.com
                serviceName: gateway
                servicePort: 8080
                path: /
    ```

```sh
$ helm upgrade openfaas \
    --namespace openfaas \
    --reuse-values \
    --values tls.yml \
    openfaas/openfaas
```

## Create a certificate

Finally, we can create the Certificate resource which triggers the actual creation of the certificate by `cert-manager`, edit the file below and replace the text `openfaas.mydomain.com` with your address for the API Gateway.

!!! example "openfaas-crt.yaml"
    ```yaml
    apiVersion: certmanager.k8s.io/v1alpha1
    kind: Certificate
    metadata:
      name: openfaas-crt
    spec:
      secretName: openfaas-crt
      dnsNames:
        - openfaas.mydomain.com
      acme:
        config:
          - http01:
              ingressClass: nginx
            domains:
              - openfaas.mydomain.com
      issuerRef:
        name: letsencrypt-staging
        kind: Issuer
    ```

```sh
$ kubectl apply -f openfaas-crt.yaml
```

You can validate that certificate has been obtained successfully using

```sh
$ kubectl describe certificate openfaas-crt
```

If it was successful you can change to the production Let's Encrypt issuer by replacing `letsencrypt-staging` with `letsencrypt-prod` in the Certificate object in `openfaas-crt.yaml` and re-run `kubectl apply -f openfaas-crt.yaml`.

## Deploy and Invoke a function

In your projects containing OpenFaaS functions, you can now deploy using your domain as the gateway, replace `openfaas.mydomain.com` with your domain as well as adding the username and password you created when you deployed OpenFaaS.

```sh
faas-cli login --gateway https://openfaas.mydomain.com --username <username> --password <password>
faas-cli deploy --gateway https://openfaas.mydomain.com
```

## Verify and Debug

There are several commands we can use to verify that the required kubernetes objects have been created.

- To check that the cert-manager issuers were created:
```sh
$ kubectl -n openfaas get issuer letsencrypt-prod letsencrypt-staging
```

- To check that your certificate was created and that cert-manager created the required secret with the actual ssl certificate:
```sh
$ kubectl -n openfaas get certificate,secret openfaas-crt
```

- If you want to tail the Nginx logs, you can use
```sh
$ kubectl logs -f $(kubectl get po -l "app=nginxingress,component=controller" -o jsonpath="{.items[0].metadata.name}")
```

## Profit!

[k8s-rbac]: https://kubernetes.io/docs/reference/access-authn-authz/rbac/
[helm]: https://helm.sh
[helm-install]: https://github.com/helm/helm/blob/master/docs/install.md
[nginx-configuration]: https://github.com/helm/charts/tree/master/stable/nginx-ingress#configuration
[openfaas-helm]: https://docs.openfaas.com/deployment/kubernetes/#20a-deploy-with-helm
[cert-manager]: https://github.com/jetstack/cert-manager
[nginx-ingress]: https://github.com/kubernetes/ingress-nginx
