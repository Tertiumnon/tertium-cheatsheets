# NextJS

## Configure

### Nginx reverse proxy

```
server {

    server_name YOUR-SERVER-NAME;
    listen 80;

    location / {
        # default port, could be changed if you use next with custom server
        proxy_pass http://localhost:3000;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;

        # if you have try_files like this, remove it from our block
        # otherwise next app will not work properly
        # try_files $uri $uri/ =404;
    }

}
```

### Start NextJS with PM2

```bash
pm2 start npm --name "{your-name}" -- start
```
