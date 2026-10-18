# Setup Nginx for Jira

## Setup DNS

### Update DNS records

| Type  | Name            | Content         |
| ----- | --------------- | --------------- |
| CNAME | www             | {server name} |
| A     | {server name} | {server IP}   |
| A     | \*              | {server IP}   |
| A     | {subdomain}   | {server IP}   |

## Setup Jira

### Update Jira connector configuration

- Stop Jira service

```bash
sudo /etc/init.d/jira stop
```

- Open `{Jira-Install}/conf/server.xml`

```bash
sudo nano /opt/atlassian/jira/conf/server.xml
```

- Comment default configuration and uncomment HTTP configuration. Add parameter `path=""`.

```xml
<Connector port="8080" relaxedPathChars="[]|" relaxedQueryChars="[]|{}^&#x5c;&#x60;&quot;&lt;&gt;"
            maxThreads="150" minSpareThreads="25" connectionTimeout="20000" enableLookups="false"
            maxHttpHeaderSize="8192" protocol="HTTP/1.1" useBodyEncodingForURI="true" redirectPort="8443"
            acceptCount="100" disableUploadTimeout="true" bindOnInit="false" scheme="http"
            path="" proxyName="{server}" proxyPort="80"/>
```

- Start Jira service

```bash
sudo /etc/init.d/jira start -fg
```

## Setup Nginx

### Create Nginx config file

```bash
sudo nano /etc/nginx/sites-available/{server}.conf
```

### Update Nginx config file

```text
server {
    server_name {server};

    location / {
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://localhost:8080;
        client_max_body_size 10M;
    }
}
```

### Enable site

```bash
sudo ln -s /etc/nginx/sites-available/{server}.conf /etc/nginx/sites-enabled/{server}.conf
sudo service nginx restart
```

###

## References

- [Configure Jira server to run behind a NGINX reverse proxy](https://confluence.atlassian.com/jirakb/configure-jira-server-to-run-behind-a-nginx-reverse-proxy-426115340.html)
