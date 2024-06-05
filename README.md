# fiware-dsc

## Getting Started

### Prerequisites

- [Debian](https://www.debian.org/) 12
- [Docker Engine](https://docs.docker.com/engine/install/debian/)
- [Docker Compose](https://docs.docker.com/compose/install/) (included with Docker Engine)
- [Certbot](https://certbot.eff.org/instructions?ws=other&os=debianbuster)
- wget `sudo apt install wget`

### Create environment variables

- Create and edit the environment file

    ```
    cp ./.env.example ./.env
    ```

### Generate the DID

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

- Update the `DID` environment variable of the **./.env** file

### Consumer

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

### Keycloak configuration

    - Create the **./config/keycloak/realms/vc-issuer.json** realm file

        ```
        cp ./config/keycloak/realms/vc-issuer.json.example ./config/keycloak/realms/vc-issuer.json
        ```

    - Edit the following fields of the **./config/keycloak/realms/vc-issuer.json** realm file

        - clients[5].attributes.vc_gx:legalName (line 767)
        - clients[5].attributes.vc_subjectDid (line 768)

    - Login to the Keyrock administration console

        ![](images/0cad6118-1ccd-4be2-82f5-667012fd00a0.png)

        ![](images/2681a735-c085-460f-ad09-4ab41fb7d204.png)

    - Create a realm **vc-issuer**

        ![](images/d5546e60-24d3-438a-ac86-cac40436e119.png)

        ![](images/640a1618-23a7-4d1d-bf33-af4e34a2dd5f.png)

    - Go to the **Users** tab

        ![](images/8470f80f-2dec-465f-a36c-503083adedfb.png)

    - Create an **admin** user

        ![](images/220ce34d-6e04-4b91-848b-ff6401adf506.png)

        ![](images/0c6c93f4-d947-47dc-8fad-b3429ffb24a2.png)

        > It is important to fill in all fields (email, first name, last name...)

    - Go to the **Credentials** tab

        ![](images/8ea085a7-dda8-4ecb-83dd-93601e1e492f.png)

    - Set a password

        ![](images/9129f86e-021c-4fa9-94ce-cd2fa7c3c761.png)

        ![](images/377e9176-3cbc-4b36-bb08-08c34e4e01ad.png)

        ![](images/acffea80-8f8d-4bb7-bdeb-1ecb7f6b68af.png)

    - Go to the **Role mapping** tab

        ![](images/a118ec77-1c87-43f2-823b-2c61e0f49b9a.png)

    - Click the **Assign role** button

        ![](images/af9c6b74-0587-4731-bfd2-6ce051b25ef2.png)

    - Filter the roles by clients

        ![](images/2ad3a207-7e70-4835-87bd-9f5a81f51e9b.png)

    - Assin the **LEGAL_REPRESENTATIVE** role

        ![](images/75d97aa9-4aad-42b2-aa0c-704b4c81c35f.png)

### Onboarding

    - From the phone, go to https://demo-wallet.fiware.dev/

    - From the computer, go to https://keycloak.example.com/realms/vc-issuer/account/

    - Go to **Verifiable Credentials**

        ![](images/e156a8a8-5a3a-4ada-8c01-0777bea4cdf9.png)

    - Login using the **vc-issuer** realm **admin** user

        ![](images/2681a735-c085-460f-ad09-4ab41fb7d204.png)

    - Select **GaiaXParticipantCredential ldp_vc** in the drop-down list

        ![](images/99a012e8-532a-460c-b215-54782eb66c7b.png)

    - Click on the **Initiate Credential-Issuance(OIDC4CI)** button

        ![](images/ada8e092-3041-499b-8b51-12fa6ef942f5.png)

        > If an error occurs, refresh the page and try again.

    - Click on the **Scan QR** button in the wallet

    - Scan the QR code

    - Click on the **Save** button

    - Click on **Get Compliancy Credential** at the bottom of the page

    - Click on **FIWARE Compliance Service**

    - Click on the **Home** button

    - Select **NaturalPersonCredential ldp_vc** and click on the **Initiate Credential-Issuance(OIDC4CI)** button

        ![](images/4d5d4a1e-c45e-4065-95ad-9fdc6db43461.png)

    - Scan and save this second QR code in the wallet

        > You should now have 3 verifiable credentials in your wallet.

    - Go to the [OnBoarding Portal](https://onboarding-portal.dsba.fiware.dev/)

    - Click on **Login with VC**

        ![](images/d6e439dc-94fe-48bf-b824-f21f746f863d.png)

    - Scan the QR code with the wallet

    - Click on the **Send Credential** button

    - Click on the **+** button
