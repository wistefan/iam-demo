# Bridging the Gap: OpenID4VC as bridge to Verifiable-Credential based Identity & Access Management 

This demo will show how an application, secured by a standard OIDC setup, can be migrated with minimal effort to a Verifiable Credentials based IAM.
[Grafana](https://grafana.com/) is used as an example application here, initially being configured to use standard OAuth2 together with Keycloak. The demo will show the required steps to change the configuration of Grafana and Keycloak, in order to provide the same login capabilities using OpenID for Verifiable Credentials.

## Local preparation

Setup the local cluster:

```sh
  cd base-cluster
  mvn clean deploy
  export KUBECONFIG=$(pwd)/target/k3s.yaml

  # enable storage
  kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.30/deploy/local-path-storage.yaml
```



## Keycloak and DID

Install keycloak and in order to prepare already for the Verifiable Credentials usage the did-helper:

```sh
helm install did-helper fiware/did-helper -f did-values.yaml
```

```sh
helm install keycloak oci://registry-1.docker.io/bitnamicharts/keycloak -f keycloak-values.yaml
```

## Install Grafana

```sh
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm install grafana grafana-community/grafana -f grafana-values.yaml
```

## Traditional login

Go to https://grafana.127.0.0.1.nip.io and login as "admin":"test" and "employee":"test"


## Verifier

Prepare Grafana for OID4VP:

```sh
helm repo add vc-authentication https://fiware.github.io/vc-authentication/
helm install authentication vc-authentication/vc-authentication -f values.yaml
```
-> got to [grafana-values.yaml](./base-cluster/grafana-values.yaml) and change to verifier configuration.

Configure the grafana-oauth client in the credentials-config-service:

```sh
curl  -X POST \
  'http://localhost:8081/service' \
  --header 'Accept: */*' \
  --header 'Content-Type: application/json' \
  --data-raw '{
  "id": "grafana-oauth",
  "defaultOidcScope": "openid email profile offline_access roles",
  "authorizationType": "FRONTEND_V2",
  "oidcScopes": {
    "openid email profile offline_access roles": {
      "authorizationType": "FRONTEND_V2",
      "credentials":{
        "type": "UserCredential",
        "trustedParticipantsLists": [],
        "trustedIssuersLists": [
          "http://trusted-issuers-list:8080"
        ],
        "holderVerification": {
          "enabled": false,
          "claim": null
        },
        "requireCompliance": false,
        "jwtInclusion": {
          "enabled": true,
          "fullInclusion": false,
          "claimsToInclude": [
            {
              "originalKey": "credentialSubject.email",
              "newKey": "email"
            },
            {
              "originalKey": "credentialSubject.firstName",
              "newKey": "name"
            },
            {
              "originalKey": "credentialSubject.firstName",
              "newKey": "displayname"
            },
            {
              "originalKey": "credentialSubject.email",
              "newKey": "login"
            },
            {
              "originalKey": "$.credentialSubject.roles[?(@.target==\"grafana-oauth\")].names[0]",
              "newKey": "roles"
            }
          ]
        } 
      },
      "presentationDefinition": {
        "id": "pd",
        "input_descriptors": [
          {
            "id": "desc",
          "constraints": {
            "fields": [
              {
                "id": "vct",
                "path": ["$.vct"],
                "filter": {
                  "const": "UserCredential"
                }
              },
              {
                "id": "subject",
                "path": ["$.firstName"]
              },
              {
                "id": "roles",
                "path": ["$.roles"]
              }
            ]
          },
          "format": {
            "vc+sd-jwt": {
              "alg": ["ES256"]
            }
          }
          }
        ]
      },
      "flatClaims": true
    }
  }
}'
```

Trust Keycloak as Issuer:

```sh
curl  -X POST \
  'http://til.127.0.0.1.nip.io:8080/issuer' \
  --header 'Accept: */*' \
  --header 'Content-Type: application/json' \
  --data-raw '{
    "did": "did:key:zDnaeVNConXu7B6xqvNSsnH9KogL4Y1uvcVuyonGiN7hPrnEA",
    "credentials": [
      {
        "credentialsType": "UserCredential"
      }  
    ]
  }'
```


## Get Credentials

### [Preparation] Setup local wallet in the Android Emulator

> :warning: This is only tested on Ubuntu. In other systems, different solutions might be required.

If running on the Mobile Phone is not possible or desirable, the Android Emulator can be used:

* install the [Android Studio](https://developer.android.com/studio/install)
* create a Virtual Device(through the device-manager), rename it to an easy typable name(in the example, its 'p-xl-4')
    * the image should support Android Version >= 13, min target SDK >= 23, for example an Google Pixel 4 image
* close Android Studio and interact directly with the emulator, its located in the Android-SDK folder(most likely something like ~/Android/Sdk/emulator)

Since retrieving and presenting Credentials requires scanning an QR-Code from the screen, the screen-output needs to be presented to the emulator camera:
* install and enable the [v4l2loopback](https://github.com/v4l2loopback/v4l2loopback) kernel module:
```shell
  sudo apt install v4l2loopback-dkms
  sudo modprobe v4l2loopback
```
* find you virtual camera:
```shell
  v4l2-ctl --list-devices
```
* stream the screen to the virtual camera, using [FFmpeg](https://ffmpeg.org/):
```shell
    sudo apt install ffmpeg
    # streams the top left 200x200px rectangle from the screen to the virtual device
    ffmpeg -f x11grab -video_size 1024x768 -i :0.0+200,200 -f v4l2 -pix_fmt yuv420p <NAME-OF-THE-CAMERA-DEVICE>
```

Once this is done, the camera can be connected to the emulator:
* check the name of the camera connected to the virtual device in the emulator:
```shell
    ~/Android/Sdk/emulator/emulator -avd p-xl-4 -webcam-list
```
* start the emulator, with the camera configured as the back-camera
```shell
  # webcam2 is the virtual camera in our example, webcam0 is any other cam in the system, p-xl-4 is the device, hardware accelaration is on
  # Vulkan is required for proper browser rendering an api >29
  ~/Android/Sdk/emulator/emulator -avd p-xl-4 -camera-back webcam2 -accel on -no-snapshot-load -feature -Vulkan -grpc 8554
```

* download the apk from https://drive.google.com/file/d/10NT5uX1uLwjI2D_XoE1eqmarv8Wo61hO/view?usp=sharing
* install the apk to the device:
```shell
    adb install app-dev-debug.apk
```
> :bulb: When first starting the app on the device, an error will be displayed. This is due to not having a Personal ID available in the demo environment and wallet and can just be ignored.


#### Get a credential as "Admin"

* Access the keycloak at ```https://keycloak.127.0.0.1.nip.io/realms/test-realm/account/oid4vci```
* Login with ```admin``` - Password: ```test```
* Open the wallet in the emulator and click on "Scan QR":
  
![Wallet - Initial Scan]
* In Keycloak, goto the "VerifiableCredentials" section and select "user-credential"
* Move the QR in the camera field and follow the flow (BE AWARE: every QR is only valid for 30s, if it times out, reload the page)
* The credential should now be shown in the "Documents" section of the app:


#### Get a credential as "Viewer"

* Access the keycloak at ```https://keycloak.127.0.0.1.nip.io/realms/test-realm/account/oid4vci```
* Login with ```employee``` - Password: ```test```
* Open the wallet in the emulator and click on "Scan QR":
  
![Wallet - Initial Scan]
* In Keycloak, goto the "VerifiableCredentials" section and select "user-credential"
* Move the QR in the camera field and follow the flow (BE AWARE: every QR is only valid for 30s, if it times out, reload the page)
* The credential should now be shown in the "Documents" section of the app:

## Login to Grafana

Open Grafana at https://grafana.127.0.0.1.nip.io. Grafana will directly redirect to the OID4VP login page.

Scan the QR with the Wallet("Authenticate" -> "Online") and select one of the credentials.