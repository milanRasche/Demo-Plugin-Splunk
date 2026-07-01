This was made for the purposes of the final Frank!Hub Demo

# Splunk Hec Plugin

A [Frank!Framework](https://frankframework.org/) plugin that adds a Log4j2 appender (`FrankSplunkAppender`) which sends log events directly to Splunk via the **HTTP Event Collector (HEC)**, instead of (or alongside) writing to local log files.

- Maven coordinates: `org.wearefrank:splunk-appender`
- Plugin class: `org.frankframework.plugins.splunk.SplunkPlugin`
- Requires: Frank!Framework 9.3.0+, Java 17
- License: Apache-2.0

## Installation

1. Build the plugin (`mvn package`) or download a release jar (see `releases.json` for which plugin version matches which Frank!Framework version).
2. Drop the resulting `splunk-appender-<version>-plugin.jar` into your Frank!Framework instance's `/opt/frank/plugins` directory.
3. On startup, `SplunkPlugin` automatically appends `log4j2-to-splunk.xml` to your `log4j.configurationFile` property and reconfigures Log4j2 — you don't need to reference the file yourself, but it must be resolvable on the plugin's classpath. In most setups you'll still want to be explicit:

```properties
log4j.configurationFile=log4j4ibis.xml,log4j2-to-splunk.xml
```

## Configuration

The plugin ships with `log4j2-to-splunk.xml`, which defines six `FrankSplunkAppender` instances wired to the standard Frank!Framework loggers, each sending to Splunk with its own `source`:

| Appender name | Logger | source |
|---|---|---|
| `splunk-hec-application` | `APPLICATION` / Root | `${instance.name}.log` |
| `splunk-hec-messages` | `MSG` | `${instance.name}-messages.log` |
| `splunk-hec-security` | `SEC` | `${instance.name}-security.log` |
| `splunk-hec-heartbeat` | `HEARTBEAT` | `${instance.name}-heartbeat.log` |
| `splunk-hec-config` | `CONFIG` | `${instance.name}-config.log` |
| `splunk-hec-ladybug` | `nl.nn.testtool`, `nl.nn.xmldecoder`, `org.frankframework.ibistesttool` | `testtool4${instance.name}.log` |

All of them read from the same set of Frank!Framework properties (`${ff:...}`), so you only need to set the values below **once** for all six appenders to pick them up.

### Properties that should be set

```properties
## Hostname to send HTTP events to
url=https://splunk:8088

## Token to authenticate with
token=${credential:username:token}

## Event index
index=

## Event sourcetype
sourcetype=
```

### Properties that may be overwritten

```properties
## Default: logger type, cannot be changed
source=

## Event 'messageFormat'
messageFormat=text

## Event host
host=$HOSTNAME

## Used for setup/debugging
ignoreExceptions=true

## Must be either sequential or parallel
send_mode=sequential

## Verify SSL connection
disableCertificateValidation=false

## When sending in batches, limit by size
batch_size_bytes=0

## When sending in batches, limit by events
batch_size_count=0

## When sending in batches, limit by time/frequency
batch_interval=0
```

> `source` is fixed per appender (see table above) — it identifies which Frank!Framework log stream an event came from, so it isn't meant to be overridden via a property.

### Additional appender attributes

The underlying `FrankSplunkAppender` (see `SplunkHecAppender.java`) supports a few more attributes than are wired up by default in `log4j2-to-splunk.xml`, in case you build a custom appender configuration:

| Attribute | Description |
|---|---|
| `channel` | Optional HEC channel GUID, used for indexer acknowledgement. |
| `type` | Sender type passed to the underlying Splunk HEC client. |
| `env` | Free-text environment tag (e.g. `dtap.stage`) included in event metadata. |
| `retries_on_error` | Number of retry attempts on a failed send (default `0`, i.e. no retries). |
| `middleware` | Fully-qualified class name of a custom `HttpSenderMiddleware` implementation to plug into the sender pipeline. |
| `eventBodySerializer` | Fully-qualified class name of an `EventBodySerializer`. The plugin ships `org.frankframework.plugins.splunk.RawEventBodySerializer`, which sends the raw formatted log line instead of wrapping it in JSON — this is what `log4j2-to-splunk.xml` uses. |
| `eventHeaderSerializer` | Fully-qualified class name of a custom `EventHeaderSerializer`. |
| `includeLoggerName` / `includeThreadName` / `includeMDC` / `includeException` / `includeMarker` | Toggle inclusion of the logger name, thread name, MDC context, exception message, and Log4j2 marker in each event. All default to `true`. |
| `connect_timeout` / `call_timeout` / `read_timeout` / `write_timeout` | HTTP timeout tuning (milliseconds), passed straight to the Splunk Java logging library's `TimeoutSettings`. |

## Credentials

`token` may be retrieved from the Frank!Framework `CredentialProvider` (such as a WebSphere AuthAlias) using the following syntax:

```properties
token=${credential:username:authaliasNAME}
```

where `authaliasNAME` is the name of the authentication alias.

## Example values

```properties
url=https://splunk:8088
token=${credential:username:token}
disableCertificateValidation=true

index=ibis_common
sourcetype=ibis:${instance.name}
```

## Running the demo locally

The repo includes a `docker-compose.yml` that spins up a Frank!Framework instance with the plugin mounted, plus a local Splunk container with HEC enabled on port `8088`:

```yaml
services:
  frankframework:
    image: nexus.frankframework.org/frankframework:latest
    ports:
      - "8080:8080"
      - "8001:8001"
    volumes:
      - ./target/splunk-appender-2.0.0-SNAPSHOT-plugin.jar:/opt/frank/plugins/splunk-plugin.jar
      - ./src/test/resources/log4j4ibis.properties:/opt/frank/resources/log4j4ibis.properties
      - ./src/main/secrets:/opt/frank/secrets
    environment:
      log4j.configurationFile: log4j4ibis.xml,log4j2-to-splunk.xml
      credentialFactory.class: nl.nn.credentialprovider.PropertyFileCredentialFactory
      credentialFactory.map.properties: /opt/frank/secrets/credentials.properties

  splunk:
    image: splunk/splunk:latest
    ports:
      - "8000:8000"
      - "8088:8088"
    environment:
      - SPLUNK_START_ARGS=--accept-license
      - SPLUNK_HEC_TOKEN=abcd1234
      - SPLUNK_PASSWORD=Passw0rd
```

1. Build the plugin jar with `mvn package`.
2. `docker compose up`.
3. Frank!Framework is reachable on `http://localhost:8080`, Splunk's web UI on `http://localhost:8000` (login `admin` / `Passw0rd`).
4. Trigger some Frank!Framework activity — logs should start appearing in Splunk under the `ibis_common` index (or whatever `index` you configured), sourcetype `ibis:<instance.name>`.

The demo's HEC token (`abcd1234`) is wired up via `src/main/secrets/credentials.properties` (`token/username=abcd1234`) and picked up through `token=${credential:username:token}`.
