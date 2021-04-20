## Deploy
There are four primary components involved in deployment of this web application:
1. the application code base,
2. nginx web server,
3. the web server gateway interface, and
4. systemd unit files

This guide has been dividing into sections representing these four components. The target system for this deployment guide is debian stable.

There are several assumptions made herein:
- application codebase install directory: /opt/ktyper/
- application virtual environment directory: /opt/ktyper/venv/
- domain name: ktyper.pasteur.fr
- ssl certificate path: /etc/letsencrypt/live/pasteur.fr/fullchain.pem
- ssl certificate key path: /etc/letsencrypt/live/pasteur.fr/privkey.pem
- ssh dhparam path: /etc/ssl/certs/dhparam.pem
- software: Python3 is installed and is the system default
- current user: user is not root, has sudo access, is in www-data group


### Application codebase
The web application requires npm for frontend compilation, sqlite3 for the database, and git to retrieve the source:
```bash
sudo apt-get install git sqlite3 npm
```

Clone repo and temporarily change owner so we can easily configure:
```bash
# NOTE: this is assumed to be cloned into /opt/
sudo git clone https://github.com/scwatts/ktyper.git -b deploy_pasteur && cd ktyper/
sudo chown $(whoami) -R ./
```

Create secret:
```bash
SECRET=$(tr -dc '[:alnum:]' < /dev/urandom | head -c${1:-64})
echo -e "{\n  \"SECRET_KEY\": \"${SECRET}\"\n}" > config/secrets.json
```

Compile the frontend:
```bash
cd frontend/
npm install
npm run build
cd ../
```

Create upload directory and venv:
```bash
mkdir -p uploads/
python3 -m venv venv/
. venv/bin/activate
```

Install necessary packages into virtual environment:
```bash
pip3 install \
    git+https://github.com/scwatts/spectracl.git \
    numpy \
    pandas \
    scipy \
    scikit_learn \
    django \
    djangorestframework \
    huey
```

Create database from migrations (ensuring virtual environment is still activate):
```bash
./manage.py makemigrations backend
./manage.py migrate
```

Deactivate virtual environment:
```bash
deactivate
```

Set permissions for codebase:
```bash
sudo chown root:www-data -R ./
sudo chmod ug+r-w,o-rwx -R ./
sudo chmod ug+w uploads/ huey.db* db.sqlite3 ./
```


### nginx server
Install nginx if not already present:
```bash
sudo apt-get install nginx
```

Copy configuration from `./config/etc/nginx/` to `/etc/nginx/`. Check that appropriate ports are open:
```bash
# List current rules in INPUT chain
sudo iptables -L INPUT

# Add new rules if required
sudo iptables -A INPUT -p tcp -m tcp --dport 80 -m state --state NEW -j ACCEPT
sudo iptables -A INPUT -p tcp -m tcp --dport 443 -m state --state NEW -j ACCEPT
```

Restart nginx server to load new config:
```bash
sudo systemctl restart nginx
```


### uWSGI
Install uWSGI and required plugins if needed:
```bash
sudo apt-get install uwsgi uwsgi-plugin-python3 python3-django-uwsgi
```

Copy configuration from `./config/etc/uwsgi/` to `/etc/uwsgi/`.


### systemd
Copy configuration from `./config/etc/systemd/` to `/etc/systemd/`. Next, load the new unit files into memory:
```bash
sudo systemctl daemon-reload
```

Enable and start unit files:
```bash
sudo systemctl enable --now ktyper-uwsgi ktyper-queuer
```

### Other
When deviating from assumptions layed out in the this section ensure that:
* the configs for nginx, uwsgi, and/or systemd unit files are updated appropriately,
* `ALLOWED_HOSTS` in ktyper/config/settings.py is set to contain the correct domain, and
* the `spectracl` executable called in ktyper/backend/tasks.py points to the correct location


## Updating model
Currently the most robust way to update a model is to do so through `spectracl`. Specifically, update the model
in the `spectracl` GitHub repo and then reinstall `spectracl` into the virtual environment:

```bash
. /opt/ktyper/venv/bin/activate
pip3 install --force-reinstall git+https://github.com/scwatts/ktyper.git
```

Following this all services should be restarted to ensure the update is propogated:
```bash
sudo systemctl restart nginx ktyper-uwsgi ktyper-queuer
```
