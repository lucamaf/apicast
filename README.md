# Customizing APIcast for Mutual SSL authentication and Basic Auth against Backend API (or Upstream in NGINX terminology)

## Adding Basic Auth
Basic Auth will just be an additional header value towards the Backend API
First hash (Base64) the value of username:password and then you can add that to a header
```
Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=
```

APIcast by default will read all `.conf` files in the `apicast.d` folder (and `location.d` subfolder) inside its prefix as part of the APIcast server configuration
So just need to add the following line to an additional file (`srv_custom.conf`) to enable BASIC AUTH:
```
proxy_set_header Authorization "Basic dXNlcm5hbWU6cGFzc3dvcmQ=;
```

## Enabling Server SSL communication
### Create public and private key for APICAST
Execute the following command and create the public and private key by providing necessary information (remember this is self-signed so not for Production)
```
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout nginx.key -out nginx.crt
```
This will create nginx.key (private key) and nginx.crt (public key) files.

### Add certificate to APICAST
Create a directory called `ssl` under `apicast` directory and copy the 2 files

### Map the certificates
Add an additional conf file (`ssl.conf`) under `apicast.d` directory which will be mapped inside the server section of the configuration
with the following content
```
listen 8443 ssl;
ssl_certificate ../ssl/nginx.crt;
ssl_certificate_key ../ssl/nginx.key;
```

## Enabling Client side SSL communication
We can also enable the client side SSL communication towards the API Backend to guarantee additional security between the Service and the gateway and prevent any MITM attack
This will result in the end in a E2E encryption of communication between the Backend API and the User App
We are going to assume that you have already the client certificate and cert under hand

Create a directory under `apicast` folder with the 2 files (`mssl` folder in this example)
Add the following lines to the previous file to send the certificates in the request towards Upstream server (`srv_custom.conf`):
```
proxy_ssl_certificate ../mssl/cert.pem;
proxy_ssl_certificate_key ../mssl/key_94744bc5-e945-457f-ad19-c98d1fe24c6f.pem;
```

## Starting APICAST Native
Now you can start APICAST (native) with the whole set of additional files:
```
THREESCALE_PORTAL_ENDPOINT=https://<access-token>@test-admin.3scale.net APICAST_LOG_FILE=logs/error.log APICAST_MANAGEMENT_API=debug APICAST_RESPONSE_CODES=true APICAST_CONFIGURATION_LOADER=boot bin/apicast -d -v -v -v -i 30 -e production
```

## Testing
```
curl --request GET \
  --url https://<apicast-production-endpoint>:8443/aValidPath \
  --header 'user-key: <valid-user-key>'
```

## References:
https://medium.com/@Jenananthan/nginx-mutual-ssl-one-way-ssl-with-multiple-clients-ae87b3de0935
https://www.nginx.com/resources/admin-guide/nginx-tcp-ssl-upstreams/

http://apiman.io/blog/gateway/security/mutual-auth/ssl/mtls/1.2.x/2016/01/22/mtls-mutual-auth-redux.html
