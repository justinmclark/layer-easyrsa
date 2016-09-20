# EasyRSA

This charm delivers the EasyRSA application to act as a Certificate Authority
(CA) and creates certificates for related charms.

EasyRSA is a command line utility to build and manage Public Key 
Infrastructure (PKI) Certificate Authority (CA).

The purpose of a Public Key Infrastructure (PKI) is to facilitate the secure
electronic transfer of information.

## Deployment
You can deploy an EasyRSA charm with Juju:

```
juju deploy easyrsa
juju deploy tls-client
juju add-relation easyrsa tls-client
```

## Using the easyrsa charm

The easyrsa charm will become a Certificate Authority (CA) and generate a CA
certificate. Other charms need only to relate to easyrsa with a requires 
using the `tls-certificates` interface.

To get a server certificate from easyrsa, the charm must include the 
`interface:tls-certificates` interface in the `layer.yaml` file. The charm must
also require the `tls` interface, in the `metadata.yaml`. The relation name may
be named what ever you wish, assume the relation is named "certificates" for 
these examples.

### CA

The interface will generate a CA certificate immediately. If another charm 
requires a CA certificate the code must react to the flag
`certificates.ca.available`. The relationship object has a method named 
`get_ca` which returns the CA certificate.

```python
@when('certificates.ca.available')
def store_ca(tls):
    '''Read the certificate authority from the relation object and install it
    on this system.'''
    # Get the CA from the relationship object.
    ca_cert = tls.get_ca()
    write_file('/usr/local/share/ca-certificates/easyrsa.crt', ca_cert)
```

### Client certificate and key

The easyrsa charm generates a client certificate after the CA certificate is 
created. If another charm needs the CA the code must react to the flag
`certificates.client.cert.available`.  The relationship object has a method 
that returns the client cert and client key called `get_client_cert`.

```python
@when('certificates.client.cert.available')
def store_client(tls):
    '''Read the client certificate from the relation object and install it on
    this system.'''
    client_cert, client_key = tls.get_client_cert()
    write_file('/home/ubuntu/client.crt', client_cert)
    write_file('/home/ubuntu/client.key', client_key)
```

### Request a server certificate

The interface will set `certificates.available` flag on a relation. The
reactive code should send three values on the relation to request a 
certificate. Call the `request_server_cert` method on the relationship object. 
The three values are: Common Name (CN), a list of Subject Alt Names (SANs), and
the file name of the certificate (the unit name with the  '/' replaced with an
underscore). For example a client charm would send:

```python
@when('certificates.available')
def send_data(tls):
    # Use the public ip of this unit as the Common Name for the certificate.
    common_name = hookenv.unit_public_ip()
    # Get a list of Subject Alt Names for the certificate.
    sans = []
    sans.append(hookenv.unit_public_ip())
    sans.append(hookenv.unit_private_ip())
    sans.append(socket.gethostname())
    # Create a path safe name by removing path characters from the unit name.
    certificate_name = hookenv.local_unit().replace('/', '_')
    # Send the information on the relation object.
    tls.request_server_cert(common_name, sans, certificate_name)
```

### Server certificate and key

The easyrsa charm generates the server certificate and key after the request 
have been made. If another charm needs the server certificate the code must 
react to the flag `{relation_name}.server.cert.available`.  The relationship 
object has a method that returns the server cert and server key called 
`get_server_cert`.

```python
@when('certificates.server.cert.available')
def store_server(tls):
    '''Read the server certificate from the relation object and install it on
    this system.'''
    server_cert, server_key = tls.get_server_cert()
    write_file('/home/ubuntu/server.cert', server_cert)
    write_file('/home/ubuntu/server.key', server_key)
```

## Contact

 * Author: Matthew Bruzek &lt;Matthew.Bruzek@canonical.com&gt;
 * Contributor: Cory Johns &lt;Cory.Johns@canonical.com&gt;
