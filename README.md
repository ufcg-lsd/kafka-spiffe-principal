## Kafka SPIFFE Principal Builder

A custom `KafkaPrincipalBuilder` implementation for Apache Kafka.
This class and documentation deals only with `SslAuthenticationContext`, we do not support any other context at the moment (Kerberos, SASL, Oauth)

#### Default behavior
The default `DefaultKafkaPrincipalBuilder` class that comes with Apache Kafka builds a principal
name according to the x509 Subject in the SSL certificate. Since there is no logic that deals with *Subject Alternative Name*,
this approach cannot handle a *SPIFFE ID*.

#### New behavior
The principal builder first looks for any valid *SPIFFE ID* in the certificate, if found, the *KafkaPrincipal* that will
be returned would be seen by an *ACL Authorizer* as **SPIFFE:spiffe://some.spiffe.id.uri**. If that fails, a normal usage of the Subject will
used with a normal **USER:CN=...**

## Installation

### Prerequesites

* You'll need to employ Maven, for installation instructions check the [Maven official page](https://maven.apache.org/install.html).
* You'll need a deployed SPIRE Server and Agent, for installation instructions check Spiffe pages on the [SPIRE Agent](https://spiffe.io/docs/latest/deploying/install-agents/) and the [SPIRE Server](https://spiffe.io/docs/latest/deploying/install-server/)  

### Build the project

```bash
mvn package
```   

This creates the file "target/kafka-spiffe-principal-X.X.X.jar" where "X.X.X" will be the actual version. Later we'll need to add this file to the JVM classpath.

### Generate SVIDs for Server 

* **OBS: The CN in the server certificate must match the machine's hostname.** 

```bash
cd /path/to/spire/

mkdir -p ./certs-server/

bin/spire-server x509 mint \
  -write certs-server -ttl 48h \
  -spiffeID "spiffe://example.org/kafka-server"

mv certs-server/bundle.pem certs-server/ca-cert.pem
mv certs-server/key.pem certs-server/kafka-server-key.pem
mv certs-server/svid.pem certs-server/kafka-server-cert.pem

# Move the files to the root of the kafka installation
cp certs-server/* /path/to/kafka/
```

### Generate SVID for client

```bash
cd /path/to/spire/

mkdir -p ./certs-client/

bin/spire-server x509 mint \
  -write ./certs-client/ -ttl 48h \
  -spiffeID "spiffe://example.org/kafka-client"

mv certs-client/bundle.pem certs-client/ca-cert.pem
mv certs-client/key.pem    certs-client/kafka-client-key.pem
mv certs-client/svid.pem   certs-client/kafka-client-cert.pem

# Move the files to the root of the kafka installation
cp certs-client/* /path/to/kafka/
```

### Configure Kafka

#### Set JVM classpath

Right before raising the Kafka broker run the following command in the same terminal:

```bash
export CLASSPATH=/path/to/kafka-spiffe-principal-X.X.X.jar
```

* **OBS**: Using `/etc/environment` or sourcing files to provide the environment variable CLASSPATH provided mixed result as far as the visibility to Kafka is concerned. We recommend using the `export` command.

#### Create keystore and trustore

* **OBS**: Throughout this section and below we use the placeholder "123456" for the password of the the keystores and truststores, replace this with your actual password in a meaningful deployment.

```bash
cd /path/to/kafka/

# Create truststore with CA file
keytool \
  -keystore truststore.jks \
  -alias CARoot \
  -importcert -file ca-cert.pem \
  -storepass 123456 \
  -noprompt

# Convert SERVER private key and certificate files into a PKCS12 file.
openssl pkcs12 \
  -export -in kafka-server-cert.pem \
  -inkey kafka-server-key.pem \
  -out kafka-server.p12 \
  -name kafka \
  -CAfile ca-cert.pem \
  -caname CARoot \
  -password pass:123456

# Import the SERVER PKCS12 file into the broker keystore
keytool \
  -importkeystore \
  -deststorepass 123456 \
  -destkeypass 123456 \
  -destkeystore broker.keystore.jks \
  -srckeystore kafka-server.p12 \
  -srcstoretype PKCS12 -srcstorepass 123456 -noprompt

# Convert CLIENT private key and certificate files into a PKCS12 file.
openssl pkcs12 \
  -export -in kafka-client-cert.pem \
  -inkey kafka-client-key.pem \
  -out kafka-client.p12 \
  -name kafka \
  -CAfile ca-cert.pem \
  -caname CARoot \
  -password pass:123456

# Import the CLIENT PKCS12 file into the CLIENT keystore
keytool \
  -importkeystore \
  -deststorepass 123456 \
  -destkeypass 123456 \
  -destkeystore client.keystore.jks \
  -srckeystore kafka-client.p12 \
  -srcstoretype PKCS12 -srcstorepass 123456 -noprompt
```


#### Kafka properties

The following configuration refers to a Kafka broker wth a TLS listener, change it as you need it. In our example our <HOSTNAME> is kafka.lsd.ufcg.edu.br

```ini
broker.id=0
port=9092
zookeeper.connect=localhost:2181
advertised.host.name=<HOSTNAME> 
offsets.topic.replication.factor=1

security.protocol=SSL
ssl.client.auth=required
security.inter.broker.protocol = SSL
ssl.enabled.protocols=TLSv1.2,TLSv1.1,TLSv1
listeners=SSL://<HOSTNAME>:9092
advertised.listeners=SSL://<HOSTNAME>:9092

# Stores
ssl.truststore.location=truststore.jks
ssl.truststore.password=123456
ssl.keystore.location=broker.keystore.jks
ssl.keystore.password=123456

# ACL related
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
principal.builder.class=io.okro.kafka.SpiffePrincipalBuilder

# Disable host name checking
ssl.endpoint.identification.algorithm=
```
