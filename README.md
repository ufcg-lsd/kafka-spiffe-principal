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

### Build the project

* **OBS**: For this section you'll need to employ Maven, for installation instructions check the [Maven official page](https://maven.apache.org/install.html).

```bash
mvn package
```   

This creates the file "target/kafka-spiffe-principal-X.X.X.jar" where "X.X.X" will be the actual version. We need to add this file to the JVM classpath.

### Configure Kafka

#### Set JVM classpath

Right before raising the Kafka broker run the following command in the same terminal:

```bash
export CLASSPATH=/path/to/kafka-spiffe-principal-X.X.X.jar
```

* **OBS**: Using `/etc/environment` or sourcing files to provide the environment variable CLASSPATH provided mixed result as far as the visibility to Kafka is concerned. We recommend using the `export` command.

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
