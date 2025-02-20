# Overview
This EST server is a fork of https://github.com/globalsign/est with two changes applied:
* removes the header/footer from the incoming CSR before performing the base64 decoding.
  This is needed for this server to work with openssl/curl and Azure IoT Edge as described in [this](https://github.com/globalsign/est/issues/23) issue
   * patch: https://github.com/arlotito/est/commit/502dbf8e4489f57774c66516aa6d13ac78135a09
* if client CAs are specified in the config file (tls->client_cas), the server will force the TLS client authentication
   * patch: https://github.com/arlotito/est/commit/8f6a83f354dd51aa700db3e45bf0b3f8b1e01f9b

## EST Server in a container
Are you looking for a containerized version of this EST server?
Here you go: https://github.com/arlotito/est-server-docker


---

Hereafter the original repo documentation.
# est

An implementation of the Enrollment over Secure Transport (EST) certificate
enrollment protocol as defined by [RFC7030](https://tools.ietf.org/html/rfc7030).

The implementation provides:

 * An EST client library;
 * An EST client command line utility using the client library; and
 * An EST server which can be used for testing and development purposes.

The implementation is intended to be mostly feature-complete, including
support for:

 * The optional `/csrattrs` and `/serverkeygen` operations, with support for
   server-generated private keys returned with or without additional
   encryption
 * The optional additional path segment
 * Optional HTTP-based client authentication on top of certificate-based
   TLS authentication

In addition, a non-standard operation is implemented enabling EST-like
enrollment using the privacy preserving protocol for distributing credentials
for keys on a Trusted Platform Module (TPM) 2.0 device, as described in Part 1,
section 24 of the Trusted Platform Module 2.0 Library specification.

## Installation

    go get github.com/arlotito/est
    go install github.com/arlotito/est/cmd/estserver
    go install github.com/arlotito/est/cmd/estclient

## Quickstart

### Starting the server

When started with no configuration file, the EST server listens on
localhost:8443 and generates a random, transient Certificate Authority (CA)
which can be used for testing:

    user@host:$ estserver &
    [1] 62405

Refer to the documentation for more details on using a configuration file.

### Getting the CA certificates

Because we're using a random, transient CA, we must retrieve the CA certificates
in insecure mode to establish an explicit trust anchor for subsequent EST
operations. Since we only need the root CA certificate to establish a trust
anchor, we use the `-rootout` flag:

    user@host:$ estclient cacerts -server localhost:8443 -insecure -rootout -out anchor.pem

We will also obtain and store the full CA certificates chain, since we'll use
it shortly to demonstrate reenrollment. Since we now have an explicit trust
anchor, we can use it instead of the `-insecure` option. Since we're storing
the full chain, we don't use the `-rootout` option here:

    user@host:$ estclient cacerts -server localhost:8443 -explicit anchor.pem -out cacerts.pem

### Enrolling with an existing private key

First we generate a new private key, here using openssl:

    user@host:$ openssl genrsa 4096 > key.pem
    Generating RSA private key, 4096 bit long modulus
    .................+++
    .............+++
    e is 65537 (0x10001)

Then we generate a PKCS#10 certificate signing request, and enroll using the
explicit trust anchor we previously obtained:

    user@host:$ estclient csr -key key.pem -cn 'John Doe' -emails 'john@doe.com' -out csr.pem
    user@host:$ estclient enroll -server localhost:8443 -explicit anchor.pem -csr csr.pem -out cert.pem

Using a configuration file, we can enroll with a private key resident on a
hardware module, such as a hardware security module (HSM) or a Trusted Platform
Module 2.0 (TPM) device. Refer to the documentation for more details.

### Enrolling with a server-generated private key

If we're unable or unwilling to create our own private key, the EST server can
generate one for us, and return it along with our certificate:

    user@host:$ estclient serverkeygen -server localhost:8443 -explicit anchor.pem -cn 'Jane Doe' -out cert.pem -keyout key.pem

Note that we can omit the `-csr` option when enrolling and the EST client can
dynamically generate a CSR for us using fields passed at the command line and
the private key we specified, or an automatically-generated ephemeral private
key if we are requesting server-side private key generation.

### Reenrolling

Whichever way we generated our private key, we can now use it to reenroll.

To reenroll a previously obtained certificate, we must use it to authenticate
ourselves during the TLS handshake with the EST server. Since our random,
transient CA uses an intermediate CA certificate, we must provide a chain of
certificates to the EST client, or the TLS handshake may fail.

Although providing the root CA certificate is optional for a TLS handshake,
the simplest option is to provide the certificate we received along with the
full chain of CA certificates which we previously obtained. To do this, we
can just append those CA certificates to the certificate we received, and
use that chain to reenroll:

    user@host:$ cat cert.pem cacerts.pem >> certs.pem
    user@host:$ estclient reenroll -server localhost:8443 -explicit anchor.pem -key key.pem -certs certs.pem -out newcert.pem

Note that when we omit the `-csr` option when reenrolling, the EST client
automatically generates a CSR for us by copying the subject field and subject
alternative name extension from the certificate we're renewing.
