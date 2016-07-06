# Automatic Certificates for Openshift Routes

It will manage all `route`s with (by default) `butter.sh/letsencrypt-managed=yes` labels.
Certificates will be stored in secrets that belong to the letsencrypt service account.


## Limitations
For now, there are the following limitations.

 * It only implements `http-01`-type verification, better known as "Well-Known".
 * Multiple domains per certificate are not supported. See issue #1.
 * It will not create the letsencrypt account.
   It needs to be created before deploying.
   See Section **Installation** below.


## Customizing

The following env variables can be used.

 * `LETSENCRYPT_ROUTE_SELECTOR` (*optional*, defaults to `butter.sh/letsencrypt-managed=yes`), to filter the routes to use;
 * `LETSENCRYPT_ALL_NAMESPACES` (*optional*, defaults to `yes`), to filter the routes to use;
 * `LETSENCRYPT_KUBE_PREFIX` (*optional*, defaults to `letsencrypt-`), the prefix for secrets and temporary routes;
 * `LETSENCRYPT_CONTACT_EMAIL` (*required for account generation*), the email that will be used by the ACME CA;
 * `LETSENCRYPT_CA` (*optional*, defaults to `https://acme-v01.api.letsencrypt.org/directory`);
 * `LETSENCRYPT_KEYTYPE` (*optional*, defaults to `rsa`), the key algorithm to use;
 * `LETSENCRYPT_KEYSIZE` (*optional*, defaults to `4096`), the size in bit for the private keys (if applicable);



## Implementation Details

### Secrets

Certificates are stored in secrets named `letsencrypt-<hostname>`, the ACME key is stored in `letsencrypt-creds`.


### Containers

The pod consists of three containers, each doing exactly one thing.
They share the filesystem `/var/www/acme-challenges` to store the challenges.

 * **Watcher Container**
   watches routes and either generates a new certificate or set the already generated certificate.

 * **Cron container**
   periodically checks whether the certificates need to be regenerated.
   When Kubernetes cron jobs are implemented, this will move outside the pod.

 * **Webserver Container**
   serves `.well-known/acme-challenge` when asking to sign the certificate.
   Uses `ibotty/s2i-nginx` on dockerhub.



## Installing Openshift-Letsencrypt

### Service Account

The "letsencrypt" service account needs to be able to manage its secrets and manage routes (modify existing, add routes and remove own).

## Notes

### HPKP

It is necessary to pin _at least_ one key to use for disaster recovery, outside the cluster!

Maybe pre-generate `n` keys and pin all of them.
On key rollover, delete the previous key, use the oldest of the remaining keys to sign the certificate, generate a new key and pin the new keys.
That way, the pin can stay valid for `(n-1)* lifetime of a key`.
That is, if no key gets compromised!

To generate the pin-sha256:

     openssl rsa -in key.pem -outform der -pubout 2>/dev/null \
       | openssl dgst -sha256 -binary | openssl enc -base64
