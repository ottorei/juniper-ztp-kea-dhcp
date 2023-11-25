# juniper-ztp-kea-dhcp
Sample configurations for Juniper ZTP with ISC Kea DHCP

## Installation
1. Install the latest Kea DHCP-server and open source hooks from the official repository.
2. The example configuration has been tested with Ubuntu 22.04 LTS and with Kea 2.4.0

## DHCP-options
1. The sample configuration takes the client's MAC-address and flexibly creates option 43 encapsulated options
2. Flex option is used to dynamically create the name of the configuration file
3. Other options are generated in client classification block such as image file name. This can not be dynamically set on Juniper devices because some models expect the name of the image file to follow certain naming conventions. Best bet is to use the official name eg. the exact same name it has on the Juniper download page, otherwise the installation process fails with varying error codes such as disk space warnings.

## Deployment
1. To use the flexible options with the provided HTTP-download, an additional software is needed which processes the HTTP-request
2. The device attempts to request config and image file with it's vendor class identifier + MAC-address from the HTTP-server
3. Correct image and config files can be served by saving the files with corresponding names to the HTTP-server
   
Dynamic serving can be achieved by passing the request to some backend application such as Django, which parses the fields from the URL. Combine Django with nginx and ¨X-Accel-Redirect¨ -header to handle the logic within Python and just serve the static files with nginx. This example uses nginx to catch the actual request, pass it to Django backend which can then figure out what to do with the device with given vendor class identifier and MAC. If Django responds to nginx's internal request with a valid ¨X-Accel-Redirect¨ -header, nginx will internally fetch that URI and serve it's contents transparently to the client.

### Example nginx configuration snippet:
```
location /ztp_fetch {
        # Include an ACL here if needed
        # include /etc/nginx/snippets/ztp_acl.conf;
        proxy_pass http://127.0.0.1:8000/device/ztp_fetch;
}

location /protected_ztp_files {
        internal;
        alias /opt/ztp/ztp;
}
```

### Example Django url path:
```
path('ztp_fetch/<mode>/<vci>/<mac>', views.ztp_fetch, name='ztp_fetch')
```

### Example Django view logic:
```
def ztp_fetch(request, mode, vci, mac):
    # Create empty response object
    response = HttpResponse()
    response['Content-Type'] = ''
    response["X-Accel-Buffering"] = "no"
    
    # Validate MAC-address
    mac = mac[0:17]
    if not re.match(r'^([0-9a-f]{2}[:-]){5}([0-9a-f]){2}$', mac, re.IGNORECASE):
        raise Exception("Invalid MAC-address")

    if mode == "conf" and mac:
        logger.info(f"Configuration file requested for VCI: {vci}, MAC: {mac}")
        response['Content-Disposition'] = f'attachment; filename={mac}.conf'
        response["X-Accel-Redirect"] = f"/protected_ztp_files/configs/{mac}"
        return response

    else:
        raise Exception("Missing or incorrect parameters")
```