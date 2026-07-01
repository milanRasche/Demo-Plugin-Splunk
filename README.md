This was made for the purposes of the final Frank!Hub Demo

# Splunk Hec Plugin
123

## 123


`log4j.configurationFile=log4j4ibis.xml,log4j2-to-splunk.xml`

The following properties should be set:
```properties
## Hostname to send HTTP events to
url=https://splunk:8088

## Token to authenticate with
token= ${credential:username:token}

## Event index
index=

## Event sourcetype
sourcetype=
```

The following properties may be overwritten:
```properties
## Default: logger type, cannot be changed
source=

## Event 'messageformat'
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

token may be retrieved from the CredentialProvider (such as an WebSphere AuthAlias) using the following syntax: `token=${credential:username:authaliasNAME}` where authaliasNAME is the name of the authentication alias.


