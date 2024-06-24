# fiware-dsc

<ul>
    <li>
        <a href="#getting-started">Getting Started</a>
        <ul>
            <li><a href="#prerequisites">Prerequisites</a></li>
            <li>
                <a href="#deployment">Deployment</a>
                <ul>
                    <li><a href="#clone-the-repository">Clone the repository</a></li>
                    <li><a href="#create-environment-variables">Create environment variables</a></li>
                    <li><a href="#generate-the-did">Generate the DID</a></li>
                    <li><a href="#consumer-configuration">Consumer configuration</a></li>
                    <li><a href="#provider-configuration">Provider configuration</a></li>
                    <li><a href="#keycloak-configuration">Keycloak configuration</a></li>
                    <li><a href="#onboarding">Onboarding</a></li>
                </ul>
            </li>
        </ul>
    </li>
    <li>
        <a href="#usage">Usage</a>
        <ul>
            <li><a href="#consumer">Consumer</a></li>
            <li><a href="#provider">Provider</a></li>
        </ul>
    </li>
</ul>

## Getting Started

### Prerequisites

- [Debian](https://www.debian.org/) 12
- [Docker Engine](https://docs.docker.com/engine/install/debian/)
- [Docker Compose](https://docs.docker.com/compose/install/) (included with Docker Engine)
- [Certbot](https://certbot.eff.org/instructions?ws=other&os=debianbuster)
- [Git](https://git-scm.com/) `sudo apt install git`
- wget `sudo apt install wget`

### Deployment

> In our example, we will use the domain name example.com. Replace it with yours wherever it appears.

#### Clone the repository

- Clone this repository

    ```
    git clone https://github.com/faubourg-numerique/fiware-dsc.git
    ```

- Access the cloned repository

    ```
    cd fiware-dsc
    ```

#### Create environment variables

- Create and edit the environment file

    ```
    cp ./.env.example ./.env
    ```

#### Generate the DID

- Create type A DNS entry for the following domain

    - example.com

- Generate HTTPS certificates

    ```
    sudo certbot certonly --standalone --key-type rsa -d example.com
    ```

- Copy certificates

    - `cp /etc/letsencrypt/live/example.com/cert.pem ./certificates/certificate.pem`
    - `cp /etc/letsencrypt/live/example.com/chain.pem ./certificates/certificate-chain.pem`
    - `cp /etc/letsencrypt/live/example.com/fullchain.pem ./certificates/full-certificate-chain.pem`
    - `cp /etc/letsencrypt/live/example.com/privkey.pem ./certificates/private-key.pem`

- Create **./tls.crt**

    ```
    cp ./certificates/full-certificate-chain.pem ./tls.crt
    ```

- Create the **./config/waltid-ssikit/signatory.conf** config file

    ```
    cp ./config/waltid-ssikit/signatory.conf.example ./config/waltid-ssikit/signatory.conf
    ```

- Run the walt.id SSI Kit

    ```
    sudo docker compose -f docker-compose-waltid-ssikit.yml up -d
    ```

- Import the private key into walt.id SSI Kit

    ```
    curl --header "Content-Type: text/plain" --data-binary "@./certificates/private-key.pem" "http://localhost:7000/v1/key/import"
    ```

- Create a web DID

    ```
    curl --location "http://localhost:7000/v1/did/create" --header "Content-Type: application/json" --data-binary @- << EOF
    {
        "method": "web",
        "keyAlias":"00000000000000000000000000000000",
        "domain": "example.com",
        "x5u": "https://example.com/tls.crt"
    }
    EOF
    ```

    > The key alias is the ID returned by walt.id SSI Kit during the previous request.

- Save the DID document

    ```
    wget -O ./did.json http://localhost:7000/v1/did/did:web:example.com
    ```

- Stop the walt.id SSI Kit

    ```
    sudo docker compose -f docker-compose-waltid-ssikit.yml down
    ```

- Add `https://example.com/tls.crt` to `verificationMethod[0].publicKeyJwk.x5u` in **did.json**

    ```json
    {
        "verificationMethod": [
            {
                "publicKeyJwk": {
                    "x5u": "https://example.com/tls.crt"
                }
            }
        ]
    }
    ```

    > You should use a [JSON formatter](https://jsonformatter.org/) to make the file readable.

- Host the following files on example.com with an HTTPS web server

    - **/tls.crt**
    - **/.well-known/did.json**

- Edit the following fields of the **./config/waltid-ssikit/signatory.conf** config file

    - proofConfig.issuerDid
    - proofConfig.issuerVerificationMethod
    - proofConfig.domain
    - proofConfig.nonce

    > The DID can be found in **./did.json**. As for the nonce, it is a unique identifier which can for example be a [UUID](https://www.uuidgenerator.net/).

- Update the `DID`, `CERTIFICATE` and `PRIVATE_KEY` environment variables of the **./.env** file

#### Consumer configuration

- Create type A DNS entries for the following subdomain

    - keycloak.example.com

- Generate HTTPS certificates

    - `sudo certbot certonly --standalone -d keycloak.example.com`

- Create and edit the **./config/nginx/nginx-consumer.conf** config file

    ```
    cp ./config/nginx/nginx-consumer.conf.example ./config/nginx/nginx-consumer.conf
    ```

- Run the consumer docker compose script

    ```
    sudo docker compose -f docker-compose-consumer.yml up -d
    ```

- [Configure Keycloak](#keycloak-configuration)

- [Perform the onboarding](#onboarding)

#### Provider configuration

- Create type A DNS entries for the following subdomain

    - keycloak.example.com
    - idm.example.com
    - as.example.com
    - vc-verifier.example.com
    - context-broker.example.com

- Generate HTTPS certificates

    - `sudo certbot certonly --standalone -d keycloak.example.com`
    - `sudo certbot certonly --standalone -d idm.example.com`
    - `sudo certbot certonly --standalone -d as.example.com`
    - `sudo certbot certonly --standalone -d vc-verifier.example.com`
    - `sudo certbot certonly --standalone -d context-broker.example.com`

- Create and edit the **./config/nginx/nginx-provider.conf** config file

    ```
    cp ./config/nginx/nginx-provider.conf.example ./config/nginx/nginx-provider.conf
    ```

- Create and edit the **./config/activation-service/as.yml** config file

    ```
    cp ./config/activation-service/as.yml.example ./config/activation-service/as.yml
    ```

- Create and edit the **./config/vc-verifier/server.yaml** config file

    ```
    cp ./config/vc-verifier/server.yaml.example ./config/vc-verifier/server.yaml
    ```

- Run the provider docker compose script

    ```
    sudo docker compose -f docker-compose-provider.yml up -d
    ```

- [Configure Keycloak](#keycloak-configuration)

- [Perform the onboarding](#onboarding)

#### Keycloak configuration

- Create the **./config/keycloak/realms/vc-issuer.json** realm file

    ```
    cp ./config/keycloak/realms/vc-issuer.json.example ./config/keycloak/realms/vc-issuer.json
    ```

- Edit the following fields of the **./config/keycloak/realms/vc-issuer.json** realm file

    - clients[5].attributes.vc_gx:legalName (line 767)
    - clients[5].attributes.vc_subjectDid (line 768)

- Login to the Keyrock administration console

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/0cad6118-1ccd-4be2-82f5-667012fd00a0.png">
    </details>

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/2681a735-c085-460f-ad09-4ab41fb7d204.png">
    </details>

- Create a realm **vc-issuer**

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/d5546e60-24d3-438a-ac86-cac40436e119.png">
    </details>

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/640a1618-23a7-4d1d-bf33-af4e34a2dd5f.png">
    </details>

- Go to the **Users** tab

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/8470f80f-2dec-465f-a36c-503083adedfb.png">
    </details>

- Create an **admin** user

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/220ce34d-6e04-4b91-848b-ff6401adf506.png">
    </details>

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/0c6c93f4-d947-47dc-8fad-b3429ffb24a2.png">
    </details>

    > It is important to fill in all fields (email, first name, last name...)

- Go to the **Credentials** tab

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/8ea085a7-dda8-4ecb-83dd-93601e1e492f.png">
    </details>

- Set a password

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/9129f86e-021c-4fa9-94ce-cd2fa7c3c761.png">
    </details>

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/377e9176-3cbc-4b36-bb08-08c34e4e01ad.png">
    </details>

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/acffea80-8f8d-4bb7-bdeb-1ecb7f6b68af.png">
    </details>

- Go to the **Role mapping** tab

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/a118ec77-1c87-43f2-823b-2c61e0f49b9a.png">
    </details>

- Click the **Assign role** button

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/af9c6b74-0587-4731-bfd2-6ce051b25ef2.png">
    </details>

- Filter the roles by clients

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/2ad3a207-7e70-4835-87bd-9f5a81f51e9b.png">
    </details>

- Assin the **LEGAL_REPRESENTATIVE** role

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/75d97aa9-4aad-42b2-aa0c-704b4c81c35f.png">
    </details>

#### Onboarding

- From the phone, go to https://demo-wallet.fiware.dev/

    <details>
        <summary>Image</summary>
        <img src="images/wallet/97d4af9e-0634-4e7f-b489-67736a2c09d9.png">
    </details>

- From the computer, go to https://keycloak.example.com/realms/vc-issuer/account/

- Go to **Verifiable Credentials**

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/e156a8a8-5a3a-4ada-8c01-0777bea4cdf9.png">
    </details>

- Login using the **vc-issuer** realm **admin** user

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/2681a735-c085-460f-ad09-4ab41fb7d204.png">
    </details>

- Select **GaiaXParticipantCredential ldp_vc** in the drop-down list

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/99a012e8-532a-460c-b215-54782eb66c7b.png">
    </details>

- Click on the **Initiate Credential-Issuance(OIDC4CI)** button

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/ada8e092-3041-499b-8b51-12fa6ef942f5.png">
    </details>

    > If an error occurs, refresh the page and try again.

- Click on the **Scan QR** button in the wallet

- Scan the QR code

- Click on the **Save** button

    <details>
        <summary>Image</summary>
        <img src="images/wallet/de69fa75-bc5c-45d1-aaa2-09994115d62c.png">
    </details>

- Click on **Get Compliancy Credential** at the bottom of the page

- Click on **FIWARE Compliance Service**

    <details>
        <summary>Image</summary>
        <img src="images/wallet/97f0f9fb-8c66-4bcf-beb6-9542617d2a5e.png">
    </details>

- Click on the **Home** button

- Select **NaturalPersonCredential ldp_vc** and click on the **Initiate Credential-Issuance(OIDC4CI)** button

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/4d5d4a1e-c45e-4065-95ad-9fdc6db43461.png">
    </details>

- Scan and save this second QR code in the wallet

    > You should now have 3 verifiable credentials in your wallet.

- Go to the [OnBoarding Portal](https://onboarding-portal.dsba.fiware.dev/)

- Click on **Login with VC**

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/d6e439dc-94fe-48bf-b824-f21f746f863d.png">
    </details>

- Scan the QR code with the wallet

- Click on the **Send Credential** button

    <details>
        <summary>Image</summary>
        <img src="images/wallet/db45e531-9933-4cfb-8571-e0cebbaf9b05.png">
    </details>

- Click on the **+** button

## Usage

> ⚠️ The user documentation is being written and is currently unfinished

- Go to [Keycloak](https://keycloak.example.com/)

- Log in to the Administration Console

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/e156a8a8-5a3a-4ada-8c01-0777bea4cdf9.png">
    </details>

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/2681a735-c085-460f-ad09-4ab41fb7d204.png">
    </details>

- Open the **vc-issuer** realm

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/5f7a79b3-d8d0-410f-9f93-971babded507.png">
    </details>

- Go to the **Users** tab

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/8470f80f-2dec-465f-a36c-503083adedfb.png">
    </details>

- Open the **admin** user details

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/a7ff5b41-5e02-493e-9d93-1b9285d10e1d.png">
    </details>

- Go to the **Role mapping** tab

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/a118ec77-1c87-43f2-823b-2c61e0f49b9a.png">
    </details>

- Click the **Assign role** button

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/af9c6b74-0587-4731-bfd2-6ce051b25ef2.png">
    </details>

- Filter the roles by clients

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/2ad3a207-7e70-4835-87bd-9f5a81f51e9b.png">
    </details>

- Assin the **customer** and or **seller** roles

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/cb8aaabb-0cdb-4e0e-971b-df9d7db0eac7.png">
    </details>

- From the computer, go to https://keycloak.example.com/realms/vc-issuer/account/

- Go to **Verifiable Credentials**

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/e156a8a8-5a3a-4ada-8c01-0777bea4cdf9.png">
    </details>

- Login using the **vc-issuer** realm **admin** user

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/2681a735-c085-460f-ad09-4ab41fb7d204.png">
    </details>

- Select **MarketplaceUserCredential ldp_vc** in the drop-down list

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/4c5ae335-9451-4bc1-9ad8-9213108af334.png">
    </details>

- Click on the **Initiate Credential-Issuance(OIDC4CI)** button

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/ada8e092-3041-499b-8b51-12fa6ef942f5.png">
    </details>

    > If an error occurs, refresh the page and try again.

- Click on the **Scan QR** button in the wallet

- Scan the QR code

- Click on the **Save** button

    <details>
        <summary>Image</summary>
        <img src="images/wallet/de69fa75-bc5c-45d1-aaa2-09994115d62c.png">
    </details>

- Go to the [DOME Marketplace](https://marketplace.dsba.fiware.dev)

- Click on the **Sign in** button

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/d645ea40-d70a-4d13-a3ab-5c6b6c08d6d1.png">
    </details>

- Select **VC Login** and click on the **Sign in* button

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/d32e4c25-ba32-4613-9c5d-e1c2dda4aabf.png">
    </details>

- Click on the admin user then click on **settings**

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/d32e4c25-ba32-4613-9c5d-e1c2dda4aabf.png">
    </details>

- Go to the **Contact mediums** tab

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/d32e4c25-ba32-4613-9c5d-e1c2dda4aabf.png">
    </details>

- Create a new billing address

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/b1e15fd8-4cdc-4b08-ab28-73d8d0952b5d.png">
    </details>

### Consumer

### Provider

- Create the following Keycloak client file **did_web_example.com.json**

    ```json
    {
        "clientId": "did:web:example.com",
        "description": "did:web:example.com",
        "surrogateAuthRequired": false,
        "enabled": true,
        "alwaysDisplayInConsole": false,
        "clientAuthenticatorType": "client-secret",
        "secret": "**********",
        "redirectUris": [],
        "webOrigins": [],
        "notBefore": 0,
        "bearerOnly": false,
        "consentRequired": false,
        "standardFlowEnabled": true,
        "implicitFlowEnabled": false,
        "directAccessGrantsEnabled": false,
        "serviceAccountsEnabled": false,
        "publicClient": false,
        "frontchannelLogout": false,
        "protocol": "SIOP-2",
        "attributes": {
            "post.logout.redirect.uris": "+",
            "client.secret.creation.time": "1675260539",
            "expiryInMin": "3600",
            "ExampleCredential_claims": "email,firstName,lastName,roles",
            "vctypes_ExampleCredential": "ldp_vc,jwt_vc_json"
        },
        "authenticationFlowBindingOverrides": {},
        "fullScopeAllowed": true,
        "nodeReRegistrationTimeout": -1,
        "defaultClientScopes": [],
        "optionalClientScopes": [],
        "access": {
            "view": true,
            "configure": true,
            "manage": true
        }
    }
    ```

- Edit `clientId`, `description`, and the keys of `attributes.ExampleCredential_claims` and `attributes.vctypes_ExampleCredential`

- Go to the **Clients** tab

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/1d4683b1-8d9e-4896-9701-dfa1f1708cb6.png">
    </details>

- Click on **Import client**

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/57e30dcb-305e-4291-98aa-1d149c119947.png">
    </details>

- Select the **did_web_example.com.json** resource file and click on the **Save** button

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/211dd4d1-8939-4d65-9316-6891c6520022.png">
    </details>

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/0c08e99b-b4e8-4b3c-aee0-f6e4ca43072e.png">
    </details>

- Go back to the **Clients** tab

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/1d4683b1-8d9e-4896-9701-dfa1f1708cb6.png">
    </details>

- Click on **did:web:example.com**

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/70d91309-f6c0-44a9-9e8e-bb7815459e82.png">
    </details>

- Go to the **Roles** tab

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/c3f083c6-89ee-469c-8c28-d7f0e7f8006a.png">
    </details>

- Click on the **Create role** button

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/fcf33d07-552a-45e0-842b-77e5bc678075.png">
    </details>

- Enter the role name (e.g. EXAMPLE) and click on the **Save** button

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/58695ba0-00fd-4abd-9f7c-ad53aa36a439.png">
    </details>

- Go to the **Users** tab

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/8470f80f-2dec-465f-a36c-503083adedfb.png">
    </details>

- Open the **admin** user details

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/a7ff5b41-5e02-493e-9d93-1b9285d10e1d.png">
    </details>

- Go to the **Role mapping** tab

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/a118ec77-1c87-43f2-823b-2c61e0f49b9a.png">
    </details>

- Click the **Assign role** button

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/af9c6b74-0587-4731-bfd2-6ce051b25ef2.png">
    </details>

- Filter the roles by clients

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/2ad3a207-7e70-4835-87bd-9f5a81f51e9b.png">
    </details>

- Select the **EXAMPLE** role and click the **Assign** button

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/05b701d6-2428-441c-b315-1397f19eabda.png">
    </details>

- Authenticate to the [DOME Marketplace](https://marketplace.dsba.fiware.dev)

- Go to the **My stock** tab

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/e94319f7-e57e-4eb3-90d6-a44b44de8e41.png">
    </details>

- Go to the **Catalogs** tab

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/e3aacce1-822d-4fe2-b8db-45e6e44f7706.png">
    </details>

- Click the **New** button

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/27abf8e7-7fca-4e40-9d28-4c6088d0ea0a.png">
    </details>

- Create a new catalog

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/e8168f72-587d-4cf9-acb1-8f29b3e58e82.png">
    </details>

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/690e68c6-eb38-474b-a034-44183b1c6703.png">
    </details>

- Select the **Launched** status and click on the **Update** button

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/51cd0d6d-48bc-4496-9f66-f5d4fd4bd476.png">
    </details>

    <details>
        <summary>Image</summary>
        <img src="images/keycloak/9d2fdae9-38de-4112-8290-84b152b0c333.png">
    </details>
