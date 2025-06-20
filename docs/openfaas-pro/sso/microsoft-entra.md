## Configure Microsoft Entra

This guide covers how to configure [Microsoft Entra]() as an identity provider for OpenFaaS IAM.

1. Sign in to [Microsoft Entra admin center](https://entra.microsoft.com/)

2. Create a new Application for OpenFaaS

    Browse to *Identity -> Applications -> Enterprise applications -> All Applications*. In the All applications pane, select *New Application*.

    This will let you browse the Microsoft Entra Gallery. Select *Create your own application*.

    Fill out the app name, select the option `Register an application to integrate with Microsoft Entra ID (App you're developing)` and click *Create*

    ![Microsoft Entra admin center, create app](/images/oidc-configuration/microsoft-entra/entra-create-app.png)

    In the next form select the account types you would want to use. We will configure the redirect URI in the next step so this field can be left out for now. Click *Register* when done.

    ![Microsoft Entra admin center, register app](/images/oidc-configuration/microsoft-entra/entra-register-app.png)

3. Configure allowed callback URL for the OpenFaaS dashboard and CLI.

    Browse to *Identity -> Applications -> App registrations*. In the All application tab select your OpenFaaS application and navigate to *Authentication*. 

    Under Platform configurations click *Add platform* and select Web.

    Enter a redirect URI:

    - `http://localhost:31111/oauth/callback` for the CLI.
    - If you are deploying the OpenFaaS dashboard, you can add the redirect URI for your dashboard e.g `https://dashboard.openfaas.example.com/auth/callback`.

    You can add more URIs later once the first one has been registered.

    Next, under Implicit grant and hybrid flows, select the `ID tokens (used for implicit and hybrid flows)` checkbox. 

    ![App registration platform configuration](/images/oidc-configuration/microsoft-entra/app-registration-platform-config.png)

4. Obtain client credentials
    
    You will need to create a client secret for the OpenFaaS app. Navigate to *Certificates and secrets* for the OpenFaaS app registration and add a new client secret. 

5. Register Microsoft Entra as a JwtIssuer with OpenFaaS

    Create a JwtIssuer object in the `openfaas` namespace to register Microsoft Entra as a trusted issuer from OpenFaaS IAM.

    The `iss` field needs to be set to the authority url of your app registration. The Authority url has the form: `https://login.microsoftonline.com/{tenant}/v2.0`.

    The `aud` field contains a set of accepted audiences. For Microsoft Entra this is the application ID of your app registration.

    Both the Directory (tenant) ID and Application (client) ID can be found in the overview of your app registration in the Microsoft Entra admin center.

    Example issuer for Entra:

    ```yaml
    apiVersion: iam.openfaas.com/v1
    kind: JwtIssuer
    metadata:
    name: login.microsoftonline.com
    namespace: openfaas
    spec:
    iss: https://login.microsoftonline.com/1fe3798478-5987-2564-b4aa-99e587365024/v2.0
    aud:
      - 068cb5cb-8cc3-4d57-8263-d6c6ce52ddff
    tokenExpiry: 12h
    ```

    The `tokenExpiry` field can be used to set the expiry time of the OpenFaaS access token.

!!! Note "SSO with the faas-cli"

    By default the faas-cli pro auth listens for OAuth callbacks on the address `http://127.0.0.1`. Entra does not support using the loopback address for redirect URIs. You need to explicitly set the flag `--redirect-host=http://localhost` to override the default value.
    
    To login with the faas-cli when using Azure Entra as the identity provider we recommend using the Implicit Id flow.

    ```sh
    faas-cli pro auth \
      --grant=implicit-id \
      --authority=https://login.microsoftonline.com/1fe3798478-5987-2564-b4aa-99e587365024/v2.0 \
      --client-id=068cb5cb-8cc3-4d57-8263-d6c6ce52ddff \
      --redirect-host=http://localhost
    ```