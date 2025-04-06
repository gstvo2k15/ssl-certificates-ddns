# ssl-certificates-ddns
Repository for generating free SSL certificates using DuckDNS, Let's Encrypt, and Certbot, supporting both individual VMs and wildcard certificates for seamless integration with a proxy manager.

1. Create python virtual-env
    ```bash
    apt install python3.12-venv -yqq
    python3.12 -m venv venv
    source venv/bin/activate
    ```

2. Install Django
    ```bash
    pip install django gunicorn
    django-admin startproject myproject
    cd ~/myproject
    gunicorn --bind 0.0.0.0:8000 myproject.wsgi:application --daemon
    ```

3. Add your VM host in `myproject/settings.py`

    `ALLOWED_HOSTS = ["yourlab.duckdns.org", "localhost", "127.0.0.1"]`

4. Install certbot and export duckdns vars

    ```bash
    sudo apt update -yqq && sudo apt install certbot -yqq

    TOKEN="YourTOKEN"
    DOMAIN="YourDNSNAME.duckdns.org"
    EMAIL="youremail@gmail.com"

    curl -s "https://www.duckdns.org/update?domains=$DOMAIN&token=$TOKEN&ip="
    OK

    sudo apt update -yqq 
    sudo apt install python3-certbot-nginx -yqq
    ```

4. Include django as reverse proxy

    ```bash
    cat <<EOF > /etc/nginx/conf.d/django
    server {
        server_name yourlab.duckdns.org;

        location / {
            proxy_pass http://127.0.0.1:8000;
            proxy_set_header Host \$host;
            proxy_set_header X-Real-IP \$remote_addr;
            proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto \$scheme;
        }

        # Managed by Certbot
        listen 443 ssl;
        ssl_certificate /etc/letsencrypt/live/yourlab.duckdns.org/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/yourlab.duckdns.org/privkey.pem;
        include /etc/letsencrypt/options-ssl-nginx.conf;
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
    }

    # Redirect HTTP to HTTPS
    server {
        listen 80;
        server_name yourlab.duckdns.org;

        location / {
            return 301 https://\$host\$request_uri;
        }
    }
    EOF

    nginx -t
    sudo systemctl restart nginx
    ```

5. Put in DMZ only for testing your local VM IP address like 192.168.1.102 or Allow Http/Https TPC/UDP connectivity to It in Router Fw rules

    ```bash
    sudo certbot --nginx -d yourlab.duckdns.org --agree-tos --email email@gmail.com

    Saving debug log to /var/log/letsencrypt/letsencrypt.log
    Requesting a certificate for yourlab.duckdns.org

    Successfully received certificate.
    Certificate is saved at: /etc/letsencrypt/live/yourlab.duckdns.org/fullchain.pem
    Key is saved at:         /etc/letsencrypt/live/yourlab.duckdns.org/privkey.pem
    This certificate expires on 2025-05-21.
    These files will be updated when the certificate renews.
    Certbot has set up a scheduled task to automatically renew this certificate in the background.

    Deploying certificate
    Successfully deployed certificate for yourlab.duckdns.org to /etc/nginx/sites-enabled/django
    Congratulations! You have successfully enabled HTTPS on https://yourlab.duckdns.org

    - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
    If you like Certbot, please consider supporting our work by:
    * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
    * Donating to EFF:                    https://eff.org/donate-le
    - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -


    ls -ltr /etc/letsencrypt/live/yourlab.duckdns.org/

    total 4
    -rw-r--r-- 1 root root 692 feb 20 10:40 README
    lrwxrwxrwx 1 root root  47 feb 20 10:40 privkey.pem -> ../../archive/yourlab.duckdns.org/privkey1.pem
    lrwxrwxrwx 1 root root  49 feb 20 10:40 fullchain.pem -> ../../archive/yourlab.duckdns.org/fullchain1.pem
    lrwxrwxrwx 1 root root  45 feb 20 10:40 chain.pem -> ../../archive/yourlab.duckdns.org/chain1.pem
    lrwxrwxrwx 1 root root  44 feb 20 10:40 cert.pem -> ../../archive/yourlab.duckdns.org/cert1.pem


    curl -I https://yourlab.duckdns.org
    bash
    HTTP/1.1 200 OK
    Server: nginx/1.18.0 (Ubuntu)
    Date: Thu, 20 Feb 2025 09:52:23 GMT
    Content-Type: text/html; charset=utf-8
    Content-Length: 12068
    Connection: keep-alive
    X-Frame-Options: DENY
    X-Content-Type-Options: nosniff
    Referrer-Policy: same-origin
    Cross-Origin-Opener-Policy: same-origin

    openssl x509 -enddate -noout -in /etc/letsencrypt/live/yourlab.duckdns.org/fullchain.pem`

    notAfter=May 21 08:42:17 2025 GMT
    ```


6. Verify in SSL labs and create crontab for schedule renewals

    https://www.ssllabs.com/ssltest/

    Introduce yourlab.duckdns.org

    Submit

    ```bash
        (crontab -l ; echo "0 3 * * * certbot renew --quiet && systemctl reload nginx") | crontab -


        crontab -l
        0 3 * * * certbot renew --quiet && systemctl reload nginx


        sudo certbot certonly --dry-run --nginx -d "$DOMAIN" --agree-tos --email "$EMAIL"
        Saving debug log to /var/log/letsencrypt/letsencrypt.log
        Account registered.
        Simulating renewal of an existing certificate for yourlab.duckdns.org
        The dry run was successful.
    ```
