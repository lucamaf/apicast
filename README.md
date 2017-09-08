# Customizing APIcast for Mutual SSL authentication and Basic Auth against Backend API (or Upstream in NGINX terminology)

## Adding Basic Auth
Basic Auth will just be an additional header value towards the Backend API
First hash (Base64) the value of ```username:password``` and then you can add that to a header
```
Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=
```

APIcast by default will read all `.conf` files in the `apicast.d` folder (and `location.d` subfolder) inside its prefix as part of the APIcast server configuration
So you just need to add the following line to an additional file (`srv_custom.conf`) to enable BASIC AUTH:
```
proxy_set_header Authorization "Basic dXNlcm5hbWU6cGFzc3dvcmQ=;
```
and place the file under:
```
apicast
  |
  |__apicast.d
        |
        |__location.d
              |
              |__srv_custom.conf
```
## Enabling Server SSL communication (APICAST to Application)
### Create public and private key for APICAST
Execute the following command and create the public and private key needed to enable SSL/TLS by providing necessary information (remember this is self-signed so not suitable for Production)
```
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout nginx.key -out nginx.crt
```
This will create nginx.key (private key) and nginx.crt (public key) files.

### Add certificate to APICAST
Create a directory called `ssl` under `apicast` directory and copy the 2 files
```
apicast
  |
  |__ssl
```
### Map the certificates
Add an additional conf file (`ssl.conf`) under `apicast.d` directory which will be mapped inside the server section of the configuration
with the following content
```
listen 8443 ssl;
ssl_certificate ../ssl/nginx.crt;
ssl_certificate_key ../ssl/nginx.key;
```

## Enabling Client side SSL communication (API Backend to APICAST)
We can also enable the client side SSL communication towards the API Backend to guarantee additional security between the Service and the gateway and prevent any MITM attack.
This will result in the end in a E2E encryption of communication between the Backend API and the User App
We are going to assume that you have already the client certificate and public key under hand

Create a directory under `apicast` folder with the 2 files (`mssl` folder in this example)
```
apicast
  |
  |__mssl
```
Add the following lines to the initial file to send the certificates in the request towards Upstream server (`srv_custom.conf`):
```
proxy_ssl_certificate ../mssl/cert.pem;
proxy_ssl_certificate_key ../mssl/key_94744bc5-e945-457f-ad19-c98d1fe24c6f.pem;
```

## Starting APICAST Native
Now you can start APICAST (native) with the whole set of additional files:
```
THREESCALE_PORTAL_ENDPOINT=https://<access-token>@test-admin.3scale.net APICAST_LOG_FILE=logs/error.log APICAST_MANAGEMENT_API=debug APICAST_RESPONSE_CODES=true APICAST_CONFIGURATION_LOADER=boot bin/apicast -d -v -v -v -i 30 -e production
```
In this case I've also decided to redirect the log, increase the debug level and update automatically the configuration every 30s.

## Testing
```
curl --request GET \
  --url https://<apicast-production-endpoint>:8443/aValidPath \
  --header 'user-key: <valid-user-key>'
```
The gateway will add Basic Auth and Client Certificate in the call towards the API Backend. You can verify some of it using for example https://requestb.in

## OpenShift deployment
With OpenShift you have to possibilites for APICAST customization:
- S2I approach
- Volume Mounting approach
We will show the second one and we will leave out the Server SSL configuration as SSL/TLS can already be terminated at the router by using OpenShift routes.

We will be using ConfigMap object (https://docs.openshift.org/latest/dev_guide/configmaps.html).
We will first create the 2 ConfigMaps that we will mount to the gateway later:
```
oc create configmap mssl --from-file=/home/centos/srv_custom.conf
oc create configmap mssld --from-file=/home/centos/mssl/
```
The first one will contain the first file we created (including Basic Auth code) and the second one the whole directory with cert and key for Client SSL Authentication.

We will then mount the 2 volumes to the DeploymentConfig
```
oc set volume dc/apicast --add --name=mssl --mount-path /opt/app-root/src/apicast.d/location.d/srv_custom.conf --source='{"configMap":{"name":"mssl","items":[{"key":"srv_custom.conf","path":"srv_custom.conf"}]}}'
oc set volume dc/apicast --add --name=mssld --mount-path /opt/app-root/src/mssl --source='{"configMap":{"name":"mssld","items":[{"key":"cert.pem","path":"cert.pem"},{"key":"key_94744bc5-e945-457f-ad19-c98d1fe24c6f.pem","path":"key_94744bc5-e945-457f-ad19-c98d1fe24c6f.pem"}]}}'
```
Once the deployment is over you can go ahead and test using the configured Route for the gateway.

## References:
https://medium.com/@Jenananthan/nginx-mutual-ssl-one-way-ssl-with-multiple-clients-ae87b3de0935
https://www.nginx.com/resources/admin-guide/nginx-tcp-ssl-upstreams/

http://apiman.io/blog/gateway/security/mutual-auth/ssl/mtls/1.2.x/2016/01/22/mtls-mutual-auth-redux.html
