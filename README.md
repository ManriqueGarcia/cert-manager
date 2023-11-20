## Utilizando el operador de cert-manager con Vault hashicorp

Este documento trata de como crear certificados desde Openshift contra un repositorio Vault de Hashi corp a través del operador de cert-manager.

Referencias: https://raylaijh.github.io/vault-certmanager-ocp/

Para ello debemos realizar los siguientes pasos

* Instalar el operador de cert-manager
* Instalar Vault
* Crear un certificado

# 1. Instalar el operador de cert-manager
Puedes ver la documentanción para instalar el operador [aquí](https://cert-manager.io/v1.2-docs/configuration/vault/)

[//]: # (sin operador)
[//]: # ($ oc create namespace cert-manager)
[//]: # ($ oc apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.2.0/cert-manager.yaml)


## Instalar el operador de Openshift desde el cli

1 . Creamos el namespace para el operador

~~~
$ oc new-project cert-manager-operator
~~~

2 . Creamos un opertor group para la instalación de los operadores.

~~~
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: cert-manager-operator
  namespace: cert-manager-operator
spec:
  targetNamespaces:
  - cert-manager-operator
  upgradeStrategy: Default
~~~

Aplicamos el operator group

~~~
oc create -f certManagerOperatorGroup.yaml
~~~

3 . Creamos la subscripción

Creamos el archivo de subscripciónn ```subscriptionOpenshiftCertManager.yaml```

~~~
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-cert-manager-operator
spec:
  channel: stable-v1
  installPlanApproval: Automatic
  name: openshift-cert-manager-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: cert-manager-operator.v1.10.3
~~~

y lo aplicamos 

~~~
$ oc create -f subscriptionOpenshiftCertManager.yaml
~~~


Esperar a que finalice la instalación, se puede ver haciendo un describe de la subscripción

~~~
$ oc describe subscription openshift-cert-manager-operator
....
  Installplan:
    API Version:  operators.coreos.com/v1alpha1
    Kind:         InstallPlan
    Name:         install-lz6g2
    Uuid:         4480a954-897b-4c2b-9b0c-36a8dabeeb1b
  Last Updated:   2023-11-20T09:23:31Z
  State:          UpgradePending
Events:           <none>
~~~

Una vez que se ha creado el operador se creará el csv, puesto que está en automático hay que esperar a que actualice hasta la última versión

~~~
$ oc get csv
NAME                            DISPLAY                                       VERSION   REPLACES                        PHASE
cert-manager-operator.v1.11.4   cert-manager Operator for Red Hat OpenShift   1.11.4    cert-manager-operator.v1.10.3   Replacing
cert-manager-operator.v1.12.0   cert-manager Operator for Red Hat OpenShift   1.12.0    cert-manager-operator.v1.11.4   Installing
~~~

~~~
$ oc get csv
NAME                            DISPLAY                                       VERSION   REPLACES                        PHASE
cert-manager-operator.v1.12.1   cert-manager Operator for Red Hat OpenShift   1.12.1    cert-manager-operator.v1.12.0   Succeeded
~~~

## Instalar Vault Hashi Corp con helm

Desde Hashicorp se recomienda que la instalación se haga mediante helm, se puede ver la documentación en [este enlace](https://developer.hashicorp.com/vault/docs/platform/k8s/helm/openshift)



1 . Añadir el repo helm de hashi corp

~~~
$ helm repo add hashicorp https://helm.releases.hashicorp.com
~~~

2 . Se actualiza el repo y se revisan versiones de vault

~~~
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "hashicorp" chart repository
...Successfully got an update from the "kyverno" chart repository
Update Complete. ⎈Happy Helming!⎈

$ helm search repo hashicorp/vault
NAME                            	CHART VERSION	APP VERSION	DESCRIPTION                          
hashicorp/vault                 	0.27.0       	1.15.2     	Official HashiCorp Vault Chart       
hashicorp/vault-secrets-operator	0.4.0        	0.4.0      	Official Vault Secrets Operator Chart
~~~

3 . Creamos un nuevo namespace para vault

~~~
$ oc create new-project vault
~~~


4 . Instalamos la última versión de Vault

~~~
helm install vault hashicorp/vault \
    --set "global.openshift=true" \
    --set "server.dev.enabled=true" \
    --set "server.image.repository=docker.io/hashicorp/vault" \
    --set "injector.image.repository=docker.io/hashicorp/vault-k8s"
    
Your release is named vault. To learn more about the release, try:

  $ helm status vault
  $ helm get manifest vault
~~~

> **NOTE**: Estos mensajes se pueden obviar

~~~
> WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: /home/mangarci/clusters/azufre/test412/auth/kubeconfig
> W1120 10:57:59.636248    7805 warnings.go:70] would violate PodSecurity "restricted:v1.24": allowPrivilegeEscalation != false (container "sidecar-injector" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "sidecar-injector" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "sidecar-injector" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "sidecar-injector" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
> W1120 10:57:59.821767    7805 warnings.go:70] would violate PodSecurity "restricted:v1.24": allowPrivilegeEscalation != false (container "vault" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "vault" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "vault" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "vault" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
~~~


Se puede comprobar que Vault crea dos pods

~~~
$ oc get pods
NAME                                    READY   STATUS    RESTARTS   AGE
vault-0                                 0/1     Running   0          13s
vault-agent-injector-575789688c-8k9l5   0/1     Running   0          13s
~~~



* El pod vault-0 ejecuta un servidor Vault en modo de desarrollo. 
* El pod vault-agent-injector realiza la inyección basándose en las anotaciones existentes o parcheadas en una implementación.


Hay que esperar hasta que los dos pods estén Ready y Running (1/1).

## Configuración de Vault

1 . Habilitamos el motor de secrets en vault

En el namespace donde hayamos instalado Vault ejecutamos.

~~~
oc exec -it vault-0 -- /bin/sh
/ $ 
~~~

Esto nos permite acceder al pod donde tenemos el servidor de vault y ahí habilitaremos la opción de pki

~~~
/ $ vault secrets enable pki
Success! Enabled the pki secrets engine at: pki/
/ $ 
~~~

> **OPCIONAL**: se puede cambiar el TTL de los certificados por defecto (30 días) de la siguiente forma

~~~
/ $ vault secrets tune -max-lease-ttl=8760h pki
~~~

2 . Generamos un certificado autofirmado interno con una rotación de 1 año (por defecto son 30 días) para nuestro dominio

~~~
/ $ vault write pki/root/generate/internal \
>     common_name=example.com \
>     ttl=8760h
~~~

3 . Configuramos los endpoints de emisión de certificados y de revocación de certificados de nuestro pki

~~~
/ $ vault write pki/config/urls \
>     issuing_certificates="http://vault.default:8200/v1/pki/ca" \
>     crl_distribution_points="http://vault.default:8200/v1/pki/crl"
Key                        Value
---                        -----
crl_distribution_points    [http://vault.default:8200/v1/pki/crl]
enable_templating          false
issuing_certificates       [http://vault.default:8200/v1/pki/ca]
ocsp_servers               []
~~~

4 . Configuramos un role que llamaremos example-dot-com que habilita la creación de certificados para el dominio example.com y sus subdominioes. Este role es un nombre lógico que mapea con la política usada para generar las credenciales.

~~~
/ $ vault write pki/roles/example-dot-com \
>     allowed_domains=example.com \
>     allow_subdomains=true \
>     max_ttl=72h
Key                                   Value
---                                   -----
allow_any_name                        false
allow_bare_domains                    false
allow_glob_domains                    false
......
~~~



5 . Ahora definiremos una política la cual mapeará todas estas rutas donde se activará Openshift. Esta política se asociará después a una service account. 

Aquí tan solo creamos la política que habilita el acceso a los secrets de nuestra PKI. 

Estas rutas permiten al token ver todos los roles creados para el  PKI y acceder a las operaciones de firma y emisión para el rol example-dot-com.

~~~
/ $ vault policy write pki - <<EOF
> path "pki*"                        { capabilities = ["read", "list"] }
> path "pki/roles/example-dot-com"   { capabilities = ["create", "update"] }
> path "pki/sign/example-dot-com"    { capabilities = ["create", "update"] }
> path "pki/issue/example-dot-com"   { capabilities = ["create"] }
> EOF
Success! Uploaded policy: pki
~~~


7 . Habilitamos en  Vault el métido de autenticación de kubernetes.

~~~
vault auth enable kubernetesvault auth enable kubernetes
/ $ vault auth enable kubernetes
Success! Enabled kubernetes auth method at: kubernetes/
~~~


8 . Configuamos el mético de autenticación de kubernetes para usar el token de la serviceaccoount, la api de kubernetes y su certificado

~~~
 vault write auth/kubernetes/config \
>     token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
>     kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
>     kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
Success! Data written to: auth/kubernetes/config
~~~


9 . Finalmente createmos un role de autenticación de kubernetes que se vincula con la política y la service account llamada issuer.
Esta issuer service account la utilizará posteriormente el cert-manager.
El role conectar la service account,issuer, con el namespace donde queramos crear los certificados con la política de Vault, en este caso testcert.
El token generado por Vault usado por la service account issuer será válido por 20 minutos.

~~~
/ $ vault write auth/kubernetes/role/issuer \
>     bound_service_account_names=issuer \
>     bound_service_account_namespaces=default \
>     policies=pki \
>     ttl=20m
Success! Data written to: auth/kubernetes/role/issuer
~~~

## ESCENARIO 1 CREAR UN ISSUER PARA PEDIR UN CERTIFICADO

1 . Creamos un nuevo proyecto donde vamos a generar nuestros certificados

~~~
$ oc new-project testcert

Now using project "testcert" on server "https://api.test412.b7lq2.azure.redhatworkshops.io:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app rails-postgresql-example

to build a new example application in Ruby. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=k8s.gcr.io/e2e-test-images/agnhost:2.33 -- /agnhost serve-hostname

~~~

2. Creamos una serviceaccount llamada issuer

~~~
$ oc create serviceaccount issuer
serviceaccount/issuer created
~~~

Al crear la service account se nos ha creado un token que utilizaremos después para crear el objeto issuer del cert-manager


$ oc get secrets
NAME                       TYPE                                  DATA   AGE
....
issuer-token-5rlnq         kubernetes.io/service-account-token   4      52s
~~~


2.Creamos un custom resource de tipo Issuer para que lo consuma el cert-manager. Creamos el fichero issuer.yaml

~~~
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: vault-issuer
  namespace: default
spec:
  vault:
    server: http://vault.vaul.svc:8200
    path: pki/sign/example-dot-com
    auth:
      kubernetes:
        mountPath: /v1/auth/kubernetes
        role: issuer
        secretRef:
          name: $ISSUER_SECRET_REF
          key: token
~~~

y lo aplicamos

~~~
$ oc create -f issuer.yaml
~~~

3 . Creamos un objeto de tipo certificado para obtener nuestro certificado con el archivo certificate.yaml

~~~
$ cat certificate.yaml 
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com
  namespace: testcert
spec:
  secretName: example-com-tls
  issuerRef:
    name: vault-issuer
  commonName: www.example.com
  dnsNames:
  - www.example.com
~~~

Creamos el objeto

~~~
oc create -f certificate.yaml
~~~

4 . Comprobar que se ha creado el certificado.

Revisamos que se nos ha creado un nuevo secret con nombre example-com-tls

~~~
$ oc get secrets -n testcert
~~~

Con un describe podemos ver si el certificado está activo y cuando expira

~~~
..........
Status:
  Conditions:
    Last Transition Time:  2023-11-20T10:50:05Z
    Message:               Certificate is up to date and has not expired
    Observed Generation:   1
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2023-11-23T10:50:05Z
  Not Before:              2023-11-20T10:49:35Z
  Renewal Time:            2023-11-22T10:49:55Z
  Revision:                1
Events:
  Type    Reason     Age   From                                       Message
  ----    ------     ----  ----                                       -------
  Normal  Issuing    41s   cert-manager-certificates-trigger          Issuing certificate as Secret does not exist
  Normal  Generated  40s   cert-manager-certificates-key-manager      Stored new private key in temporary Secret resource "example-com-rcr5s"
  Normal  Requested  40s   cert-manager-certificates-request-manager  Created new CertificateRequest resource "example-com-f6psj"
  Normal  Issuing    40s   cert-manager-certificates-issuing          The certificate has been successfully issued
~~~



Por último podmeos extraer nuestro certificado y revisar su configuración

~~~
$ oc get secrets example-com-tls -o yaml|grep tls.crt|awk '{print $2}' |base64 -d |openssl x509 -text -noout
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            48:0c:ee:ce:b0:18:5f:e0:05:77:ab:1c:1a:58:cb:c9:44:2d:b2:79
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = example.com
        Validity
            Not Before: Nov 20 10:49:35 2023 GMT
            Not After : Nov 23 10:50:05 2023 GMT
        Subject: CN = www.example.com
        Subject Public Key Info:
        ......
          X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment, Key Agreement
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Subject Key Identifier: 
                11:10:B9:68:A5:30:4D:61:C5:9D:C6:83:66:6C:56:93:C2:00:66:5D
            X509v3 Authority Key Identifier: 
                A7:3F:4C:8E:59:21:60:B0:71:A2:B4:C7:7B:44:B2:46:22:20:01:A2
            Authority Information Access: 
                CA Issuers - URI:http://vault.default:8200/v1/pki/ca
            X509v3 Subject Alternative Name: 
                DNS:www.example.com
            X509v3 CRL Distribution Points: 
                Full Name:
                  URI:http://vault.default:8200/v1/pki/crl
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
    ..............
~~~  
  