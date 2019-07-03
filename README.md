# step-sds
Secret discovery service (SDS), simplifying the certificate management.

Step SDS server provides supports both mTLS and Unix Domain Sockets
configuration, use the one that best adapts to your environment.

## mTLS initialization

### Using step-sds

The easiest way to initialize a PKI to do mTLS between between
[Envoy](https://www.envoyproxy.io) and our SDS server is to run `step-sds init`,
with the url and root certificate of an online
[certificates](https://github.com/smallstep/certificates) CA.

```sh
$ step-sds init --ca-url https://ca.smallstep.com:9000 --root ~/.step/certs/root.crt
✔ What would you like to name your new PKI? (e.g. SDS): SDS
✔ What do you want your PKI password to be? [leave empty and we'll generate one]:
✔ What address will your new SDS server listen at? (e.g. :443): :8443
✔ What DNS names or IP addresses would you like to add to your SDS server? (e.g. sds.smallstep.com[,1.1.1.1,etc.]): sds.smallstep.com
✔ What would you like to name your SDS client certificate? (e.g. envoy.smallstep.com): envoy.smallstep.com
✔ What do you want your certificates password to be? [leave empty and we'll generate one]:
✔ Key ID: jO37dtDbku-Qnabs5VR0Yw6YFFv9weA18dp3htvdEjs (mariano@smallstep.com)

✔ Root certificate: /home/user/.step/sds/root_ca.crt
✔ Root private key: /home/user/.step/sds/root_ca_key
✔ Intermediate certificate: /home/user/.step/sds/intermediate_ca.crt
✔ Intermediate private key: /home/user/.step/sds/intermediate_ca_key
✔ SDS certificate: /home/user/.step/sds/sds_server.crt
✔ SDS private key: /home/user/.step/sds/sds_server_key
✔ SDS client certificate: /home/user/.step/sds/sds_client.crt
✔ SDS client private key: /home/user/.step/sds/sds_client_key
✔ SDS configuration: /home/user/.step/config/sds.json

Your PKI is ready to go.
You can always generate new certificates or change passwords using step.
```

The `init` command will generate a root and intermediate certificate, with both
keys encrypted using the same password. And a server certificate for the
step-sds (sds_server.crt) and a client certificate for Envoy or the SDS client
(sds_client.crt), the keys of the SDS server an client will be encrypted with
another password. It will also generate an initial configuration file. All those
files will be stored in your `STEPPATH` (just run `step path` to know where).

If you want to change the passwords or create your own PKI you can always use
[step](https://github.com/smallstep/cli) for it.

### Using step

As we mention before we can use [step](https://github.com/smallstep/cli).
Assuming that the SDS is running on sds.smallstep.com and we name the envoy
client certificate as envoy.smallstep.com we can just run:

```sh
# Root and intermediate
step certificate create --profile root-ca "Smallstep SDS Root CA" root.crt root.key
step certificate create --profile intermediate-ca --ca root.crt --ca-key root.key "Smallstep SDS Intermediate CA" int.crt int.key

# Step SDS
step certificate create --profile leaf --ca int.crt --ca-key int.key --no-password --insecure --not-after 87600h sds.smallstep.com sds.pem sds.key
step certificate bundle sds.pem int.crt sds.crt

# Envoy
step certificate create --profile leaf --ca int.crt --ca-key int.key --no-password --insecure --not-after 87600h envoy.smallstep.com envoy.pem envoy.key
step certificate bundle envoy.pem int.crt envoy.crt
```

## Running the SDS server

With the PKI and configuration file created in the initialization, we can start
the server using the `step-sds run` command:

```sh
$ bin/step-sds run ~/.step/config/sds.json
Please enter the password to decrypt the provisioner key:
Please enter the password to decrypt /Users/mariano/.step/sds/sds_server_key:
INFO[0002] Serving at tcp://[::]:8443 ...                grpc.start_time="2019-04-11T19:19:37-07:00"
```

By default it ask you for the password to decrypt the provisioner key, and for
the certificate key password if this is encrypted. You can avoid the prompts
using `--password-file` and `--provisioner-password-file` flags.

```
$ bin/step-sds run ~/.step/config/sds.json --password-file /run/secrets/key.password --provisioner-password-file /run/secrets/provisioner.password
INFO[0000] Serving at tcp://[::]:8443 ...                grpc.start_time="2019-04-11T19:21:59-07:00"
```

Or you can always write them in the sds.json:

```json
{
   "network": "tcp",
   "address": ":8443",
   "root": "/home/user/.step/sds/root_ca.crt",
   "crt": "/home/user/.step/sds/sds_server.crt",
   "key": "/home/user/.step/sds/sds_server_key",
   "password": "[my-certificate-key-password]",
   "authorizedIdentity": "envoy.smallstep.com",
   "authorizedFingerprint": "8597a5d0b86f4a630f64fbb903b613ceb04756319a156bb6a6faed95394040ff",
   "provisioner": {
      "issuer": "mariano@smallstep.com",
      "kid": "jO37dtDbku-Qnabs5VR0Yw6YFFv9weA18dp3htvdEjs",
      "ca-url": "https://ca.smallstep.com:9000",
      "root": "/home/user/.step/certs/root_ca.crt",
      "password": "[my-provisioner-password]"
   },
   "logger": {
      "format": "text"
   }
}
```

And then just:

```sh
$ bin/step-sds run ~/.step/config/sds.json
INFO[0000] Serving at tcp://[::]:8443 ...                grpc.start_time="2019-04-11T19:24:09-07:00"
```

We can also configure the server using Unix domain socket, if you decide to use
this configuration the sds.json will look a little bit different as it won't be
necessary to configure the TLS certificates, and you will only need to set the
right network (unix), address (with a file path) and a provisioner configured in
your certificates CA:

```json
{
    "network": "unix",
    "address": "/tmp/sds.unix",
    "provisioner": {
       "issuer": "sds@smallstep.com",
       "kid": "oA1x2nV3yClaf2kQdPOJ_LEzTGw5ow4r2A5SWl3MfMg",
       "ca-url": "https://ca:9000",
       "root": "/home/user/.step/certs/root_ca.crt"
    },
    "logger": {
       "format": "text"
    }
 }
 ```

## Docker Compose example

In [examples/docker](examples/docker) directory we can find a docker-compose
examples that initializes a CA, a SDS server, and Envoy proxying request to two
different servers, we are naming them frontend and backend. The SDS will
generate certificates and send them to Envoy, the CommonName and DNS names of
the certificates will be indicated by the `tls_certificate_sds_secret_configs`
name in the [envoy configuration](examples/docker/envoy/server.yaml). In our
examples we are using `hello.smallstep.com` for the frontend server and
`internal.smallstep.com` for the backend server. And finally the configuration
forces the use of a client and certificate to access the backend server, this
certificate must be signed by the CA server.

To initialize the examples just run from the step-sds main directory:

```sh
make docker
cd examples/docker/
docker-compose up
```

Once everything is running we can configure our environment to do some tests,
first we'll need to add the following entries in our `/etc/hosts`.

```
127.0.0.1       ca.smallstep.com
127.0.0.1       internal.smallstep.com
127.0.0.1       hello.smallstep.com
```

Then we are going to bootstrap certificates configuration in a non-standard
STEPPATH so we don't mess with our environment:

```sh
$ export STEPPATH=/tmp
$ step ca bootstrap --ca-url https://ca.smallstep.com:9000 --fingerprint 154fa6239ba9839f50b6a17f71addb77e4c478db116a2fbb08256faa786245f5
The root certificate has been saved in /tmp/certs/root_ca.crt.
Your configuration has been saved in /tmp/config/defaults.json.
```

The we can do some tests using curl. If we don't use the root certificate we
will get the well-known error:

```sh
$ curl https://hello.smallstep.com:10000
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.haxx.se/docs/sslcerts.html

curl performs SSL certificate verification by default, using a "bundle"
 of Certificate Authority (CA) public keys (CA certs). If the default
 bundle file isn't adequate, you can specify an alternate file
 using the --cacert option.
If this HTTPS server uses a certificate signed by a CA represented in
 the bundle, the certificate verification probably failed due to a
 problem with the certificate (it might be expired, or the name might
 not match the domain name in the URL).
If you'd like to turn off curl's verification of the certificate, use
 the -k (or --insecure) option.
HTTPS-proxy has similar options --proxy-cacert and --proxy-insecure.
```

If we pass the `--cacert /tmp/certs/root_ca.crt` flag everything will work as
expected, and we'll get a response from the frontend server:

```sh
$ curl --cacert /tmp/certs/root_ca.crt https://hello.smallstep.com:10000
Hello TLS!
```

But if we try the same with the backend server we will get an error because a
mTLS connection is required:

```sh
$ curl --cacert /tmp/certs/root_ca.crt https://internal.smallstep.com:10001
curl: (35) error:1401E410:SSL routines:CONNECT_CR_FINISHED:sslv3 alert handshake failure
```

We will need to use our CA the generate some client certificates:

```sh
$ step ca certificate client.smallstep.com client.crt client.key
✔ Key ID: oA1x2nV3yClaf2kQdPOJ_LEzTGw5ow4r2A5SWl3MfMg (sds@smallstep.com)
✔ Please enter the password to decrypt the provisioner key: password
✔ CA: https://ca.smallstep.com:9000
✔ Certificate: client.crt
✔ Private Key: client.key
```

And then if we try again with curl with the certificates that we've just
generated, we will get the message from the backend server:

```sh
$ curl --cacert /tmp/certs/root_ca.crt --cert client.crt --key client.key https://internal.smallstep.com:10001
Hello mTLS!
```

The docker compose also includes the SDS server configured using unix sockets,
and we can do the same tests just replacing the ports:

```sh
$ curl --cacert /tmp/certs/root_ca.crt https://hello.smallstep.com:10010
Hello TLS!
$ curl --cacert /tmp/certs/root_ca.crt https://internal.smallstep.com:10011
curl: (35) error:1401E410:SSL routines:CONNECT_CR_FINISHED:sslv3 alert handshake failure
$ curl --cacert /tmp/certs/root_ca.crt --cert client.crt --key client.key https://internal.smallstep.com:10011
Hello mTLS!
```

# Emojivoto example

The [examples/emojivoto](examples/emojivoto) directory contains an example of
using envoy, step-sds and step-certificates on a simple microservice application
that allows users to vote for their favorite emoji, and tracks votes received on
a leaderboard. This example uses Buoyant's
[emojivoto](https://github.com/BuoyantIO/emojivoto) as its basis.

The application is composed of the following 3 services:

* emojivoto-web: Web frontend and REST API
* emojivoto-emoji-svc: gRPC API for finding and listing emoji
* emojivoto-voting-svc: gRPC API for voting and leaderboard

Besides using gRPC, the application doesn't support TLS certificates, but we're
going to use envoy and step-sds as a simple service mesh that will handle the
communications between services using (m)TLS.

In our example all the services will be behind ingress proxy and a TLS
certificate will be available for all of them. Both gRPC services will require a
client certificate from our certificate authority, so only mTLS connections will
be allowed. The web service that is the one connecting to the gRPC services will
use an egress proxy in Envoy with a client certificate, so it will be able to
connect to it.

The emojivoto example uses kubernetes, so you will need to have access to a
kubernetes cluster, if you don't
[minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/) or
[docker](https://www.docker.com) provides you with one.

to run the example you just need to run the following commands:

```sh
$ cd examples/emojivoto
$ make
kubectl apply -f ca.yaml
namespace/step created
secret/step-certificates-ca-password created
secret/step-certificates-provisioner-password created
configmap/step-certificates-config created
configmap/step-certificates-certs created
configmap/step-certificates-secrets created
service/ca created
deployment.apps/step-certificates created
sleep 2
kubectl -n step wait --for=condition=Ready -l app.kubernetes.io/name=step-certificates pod
pod/step-certificates-6fc86d5689-spzvv condition met
kubectl apply -f emojivoto.yaml
namespace/emojivoto created
serviceaccount/emoji created
serviceaccount/voting created
serviceaccount/web created
secret/step-sds-secrets created
configmap/step-sds-certs created
configmap/step-sds-config created
configmap/envoy-web-config created
configmap/envoy-emoji-config created
configmap/envoy-voting-config created
deployment.apps/emoji created
service/emoji-svc created
deployment.apps/voting created
service/voting-svc created
deployment.apps/web created
service/web-svc created
```

This will install step-certificates as a certificate authority in the step
namespace and the emojivoto services in the namespace with the same name. To
test it you will need to edit your hosts file and point `web-svc.emojivoto` to
the ClusterIP of the `web-svc` service, and then just go to
https://web-svc.emojivoto. 

The certificate of our web app is signed by our CA and you will see the unsafe
warning. If you want to avoid it you can always install the root certificate in
your environment:

```sh
$ cat <<EOF > /tmp/root_ca.crt
-----BEGIN CERTIFICATE-----
MIIBhTCCASugAwIBAgIQTiiy0M/WWuVz2cDakLykdzAKBggqhkjOPQQDAjAhMR8w
HQYDVQQDExZTbWFsbHN0ZXAgVGVzdCBSb290IENBMB4XDTE5MDcxMjIyMTQxNFoX
DTI5MDcwOTIyMTQxNFowITEfMB0GA1UEAxMWU21hbGxzdGVwIFRlc3QgUm9vdCBD
QTBZMBMGByqGSM49AgEGCCqGSM49AwEHA0IABNsTsgcRwTakVB+ouxeWzefBaLxu
hq/7d4qLbGw5pGixG0f6kN4HtIVxjZru+ABRL3PjKWUffXWiJD8XK2/QJSmjRTBD
MA4GA1UdDwEB/wQEAwIBBjASBgNVHRMBAf8ECDAGAQH/AgEBMB0GA1UdDgQWBBSF
idiUKAm0h3qnuYHq4MqgpzZsODAKBggqhkjOPQQDAgNIADBFAiEAwwKqV1AxH4ss
U69xQ6ZYIjv6l7xLWkFwDaZQXFtLsyYCIBuUpyIHlZBA0Vp5TPZgdiXIpcIrr8+z
5bpQRw86QnPY
-----END CERTIFICATE-----
EOF
$ step certificate install /tmp/root_ca.crt
Certificate /tmp/root_ca.crt has been installed.
X.509v3 Root CA Certificate (ECDSA P-256) [Serial: 1038...4951]
  Subject:     Smallstep Test Root CA
  Issuer:      Smallstep Test Root CA
  Valid from:  2019-07-12T22:14:14Z
          to:  2029-07-09T22:14:14Z
```

But remember to remove it after your test as this certificate is public and
anyone can use it:

```sh
$ step certificate uninstall /tmp/root_ca.crt
Certificate /tmp/root_ca.crt has been removed.
X.509v3 Root CA Certificate (ECDSA P-256) [Serial: 1038...4951]
  Subject:     Smallstep Test Root CA
  Issuer:      Smallstep Test Root CA
  Valid from:  2019-07-12T22:14:14Z
          to:  2029-07-09T22:14:14Z
```
