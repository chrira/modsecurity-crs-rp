# modsecurity-crs-rp

This Docker image inherits from the official OWASP Core Rule Set Docker image (ModSecurity + Core Rule Set) and adds some configurable variables and an Apache Reverse Proxy configuration.

The goal is to provide a fully functional CRS in a single command:

* This container can be placed in front of an application.
* Many CRS variables can be set.
* And it can additionally consider ModSecurity CRS tuning.

## Environment Variables

* PARANOIA: paranoia_level
* EXECUTING_PARANOIA: executing_paranoia_level
* ENFORCE_BODYPROC_URLENCODED: enforce_bodyproc_urlencoded
* ANOMALYIN: inbound_anomaly_score_threshold
* ANOMALYOUT: outbound_anomaly_score_threshold
* ALLOWED_METHODS: allowed_methods
* ALLOWED_REQUEST_CONTENT_TYPE: allowed_request_content_type
* ALLOWED_REQUEST_CONTENT_TYPE_CHARSET: allowed_request_content_type_charset
* ALLOWED_HTTP_VERSIONS: allowed_http_versions
* RESTRICTED_EXTENSIONS: restricted_extensions
* RESTRICTED_HEADERS: restricted_headers
* STATIC_EXTENSIONS: static_extensions
* MAX_NUM_ARGS: max_num_args
* ARG_NAME_LENGTH: arg_name_length
* ARG_LENGTH: arg_length
* TOTAL_ARG_LENGTH: total_arg_length
* MAX_FILE_SIZE: max_file_size
* COMBINED_FILE_SIZES: combined_file_sizes

See <https://coreruleset.org/> for further information.

* BACKEND: application backend
* PORT: listening port of apache
  
## ModSecurity Tuning

There are two possible ways to pass ModSecurity tuning rules to the container:

* To map the ModSecurity tuning file(s) via volumes into the container during the run command 
* To copy the ModSecurity tuning file(s) into the created container and then start the container

### Map ModSecurity tuning file via volume

```bash
docker run -dti \
   --name apachecrsrp \
   -p 1.2.3.4:80:8001 \
   -v /path/to/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf:/etc/apache2/modsecurity.d/owasp-crs/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf \
   -v /path/to/RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf:/etc/apache2/modsecurity.d/owasp-crs/RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf \
   franbuehler/modsecurity-crs-rp:v3.1
```

### Copy ModSecurity tuning file into created container

This example can be helpful when no volume mounts are possible (some CI pipelines).

```bash
docker create -ti --name apachecrsrp \
   -p 1.2.3.4:80:8001 \
   franbuehler/modsecurity-crs-rp:v3.1
  
docker cp /path/to/RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf \
   apachecrsrp:/etc/apache2/modsecurity.d/owasp-crs/RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf
  
docker start apachecrsrp
```

## docker run examples

### Full example with all possible environment variables

```bash
docker run -dti --name apachecrsrp -p 0.0.0.0:80:8001 \
   -e PARANOIA=1 \
   -e EXECUTING_PARANOIA=2 \
   -e ENFORCE_BODYPROC_URLENCODED=1 \
   -e ANOMALYIN=10 \
   -e ANOMALYOUT=5 \
   -e ALLOWED_METHODS="GET POST PUT" \
   -e ALLOWED_REQUEST_CONTENT_TYPE="text/xml|application/xml|text/plain" \
   -e ALLOWED_REQUEST_CONTENT_TYPE_CHARSET="utf-8|iso-8859-1" \
   -e ALLOWED_HTTP_VERSIONS="HTTP/1.1 HTTP/2 HTTP/2.0" \
   -e RESTRICTED_EXTENSIONS=".cmd/ .com/ .config/ .dll/" \
   -e RESTRICTED_HEADERS="/proxy/ /if/" \
   -e STATIC_EXTENSIONS="/.jpg/ /.jpeg/ /.png/ /.gif/" \
   -e MAX_NUM_ARGS=128 \
   -e ARG_NAME_LENGTH=50 \
   -e ARG_LENGTH=200 \
   -e TOTAL_ARG_LENGTH=6400 \
   -e MAX_FILE_SIZE=100000 \
   -e COMBINED_FILE_SIZES=1000000 \
   -e BACKEND=http://192.168.192.57:8000 \
   -e PORT=8001 \
   franbuehler/modsecurity-crs-rp
```

### Example run command for CI integration when no port mapping is possible

```bash
docker run -dt --name apachecrsrp \
   -e PARANOIA=1 \
   -e ANOMALYIN=5 \
   -e ANOMALYOUT=4 \
   -e BACKEND=http://172.17.0.1:8000 \
   -e PORT=8001 \
   --expose 8001 \
   franbuehler/modsecurity-crs-rp
```

### Just another example

```bash
docker run -dti --name apachecrsrp \
   -p 1.2.3.4:80:8080 \
   -e PARANOIA=1 \
   -e EXECUTING_PARANOIA=3 \
   -e ANOMALYIN=10 \
   -e ANOMALYOUT=5 \
   -e MAX_NUM_ARGS=255 \
   -e ARG_NAME_LENGTH=100 \
   -e ARG_LENGTH=400 \
   -e TOTAL_ARG_LENGTH=64000 \
   -e MAX_FILE_SIZE=1048576 \
   -e COMBINED_FILESIZES=1048576 \
   -e BACKEND=http://192.168.192.57:8000 \
   -e PORT=8080 franbuehler/modsecurity-crs-rp
```

## Example docker-compose.yaml

### docker-compose.yaml for easier use

See: <https://github.com/franbuehler/modsecurity-crs-rp/blob/v3.1/docker-compose.yaml>

## OpenShift configuration with SSL

To run the WAF on OpenShift, following steps are needed:

1. Provide an SSL certificate.
1. The OpenShift project needs first a configmap with the https.conf file for the reverse proxy.
1. The second step is to deploy the WAF (create OpenShift resources).

### SSL certificate for the WAF (reverse proxy)

SSL connection from the route to the WAF.

#### prepare SSL certificate

First we need an [SSL certificate](https://www.linux.com/learn/creating-self-signed-ssl-certificates-apache-linux) for the WAF.
This will be added to the OpenShift project as [secret](https://docs.openshift.com/container-platform/latest/dev_guide/secrets.html).

Copy the created cert and key into the cert-waf directory:

```bash
cp new.cert.cert resources/cert-waf/tls.crt
cp new.cert.key resources/cert-waf/tls.key
```

#### add SSL certificate as secret

Create the `certificate-waf` secret with the tls files.

```bash
oc create secret generic certificate-waf --from-file=tls.crt=resources/cert-waf/tls.crt --from-file=tls.key=resources/cert-waf/tls.key --type=kubernetes.io/tls
```

### SSL certificate for the application

SSL connection between the WAF (reverse proxy) and the application.

The cert for the communication is mounted into the WAF pod, see [secured-routes](https://docs.openshift.com/container-platform/latest/architecture/networking/routes.html#secured-routes).
This cert is already accessed inside the httpd.conf.

Explanation ::
If the destinationCACertificate field is left empty, the router automatically leverages the certificate authority that is generated for service serving certificates, and is injected into every pod as /var/run/secrets/kubernetes.io/serviceaccount/service-ca.crt. This allows new routes that leverage end-to-end encryption without having to generate a certificate for the route. This is useful for custom routers or the F5 router, which might not allow the destinationCACertificate unless the administrator has allowed it.

### provide the httpd.conf file by configmap

The httpd.conf file can be mounted into the container by adding a configmap.

With a configmap, the configuration has not to be built into the Docker image. It can easily be updated by changing the configmap.

Create httpd.conf configmap inside the OpenShift project.

```bash
oc create configmap httpd.conf --from-file=resources/configmap/httpd.conf
```

### Deploy WAF on OpenShift

The OpenShift template is provided by this repository.

Minimal parameter is the backend for the WAF (reverse proxy).
It consists of the https protocol, the service and port where the application is reachable inside the OpenShift project.

#### Create WAF resources

```bash
oc process -f resources/modsecurity-waf-template.yaml \
   -p BACKEND=https://prometheus-app:8443/ \
  | oc apply -f -
```