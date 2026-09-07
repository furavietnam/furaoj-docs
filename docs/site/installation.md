# Installing the site
(Tested on Debian 13)

## Installing the prerequisites

```shell-session
sudo apt update
sudo apt install -y git gcc g++ make python3-dev python3-pip python3-venv libxml2-dev libxslt1-dev zlib1g-dev gettext curl redis-server pkg-config zip acl postgresql libpq-dev ca-certificates
sudo curl -fsSL https://deb.nodesource.com/setup_26.x | sudo -E bash -
sudo apt-get install -y nodejs
sudo mkdir -p /mnt/FuraOJ/{contestdatacache,logs,media,static,site,problem_data,userdatacache}
sudo setfacl -R -m d:u::rwx,d:g::rwx,d:o::rwx /mnt/FuraOJ
sudo setfacl -R -m u::rwx,g::rwx,o::rwx /mnt/FuraOJ
sudo groupadd furaoj
sudo useradd -m -g furaoj -s /sbin/nologin furaoj
sudo usermod -L furaoj
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker furaoj
sudo systemctl restart docker
cd /mnt/FuraOJ/
```

## Creating the database

Next step is to set up the database itself. You should execute the commands listed below to create the necessary database and user.

```shell-session
sudo -u postgres psql
postgres> CREATE USER furaoj WITH PASSWORD '<your postgres password>';
postgres> CREATE DATABASE furaoj OWNER furaoj;
postgres> GRANT ALL PRIVILEGES ON DATABASE furaoj TO furaoj;
postgres> \q
```

## Installing prerequisites

Now that you are done, you can start installing the site. First, create a virtual environment and activate it. Here, we'll create a virtual environment named `furaojsite`.

```shell-session
python3 -m venv furaojsite
. furaojsite/bin/activate
```

You should see `(furaojsite)` prepended to your shell. Henceforth, `(furaojsite)` commands assume you are in the code directory, with the virtual environment active.

?> The virtual environment will help keep the modules needed separate from the system package manager, and save you many headaches when updating. Read more about virtual environments [here](https://docs.python.org/3/tutorial/venv.html).

Now, fetch the site source code:

```shell-session
(furaojsite) cd site
(furaojsite) git clone --recursive https://github.com/furavietnam/furaoj.git .
```

Install Python dependencies into the virtual environment.

```shell-session
(furaojsite) pip3 install -r requirements.txt
```

Install Node.js packages:

```shell-session
(furaojsite) npm install
```

You will now need to configure `dmoj/local_settings.py`. You should download the sample settings file using `curl`, open it with `nano` to make changes as necessary, and update your PostgreSQL credentials.

```shell-session
(furaojsite) curl -sSL -o dmoj/local_settings.py https://raw.githubusercontent.com/furavietnam/furaoj-docs/refs/heads/main/sample_files/local_settings.py
(furaojsite) nano dmoj/local_settings.py
```

?> Leave debug mode on for now; we'll disable it later after we've verified that the site works. <br> <br>
Generally, it's recommended that you add your settings in `dmoj/local_settings.py` rather than modifying `dmoj/settings.py` directly. `settings.py` will automatically read `local_settings.py` and load it, so write your configuration there.

## Compiling assets

FuraOJ uses `sass` and `autoprefixer` to generate the site stylesheets. FuraOJ comes with a `make_style.sh` script that may be run to compile and optimize the stylesheets.

```shell-session
(furaojsite) ./make_style.sh
```

Now, collect static files into `STATIC_ROOT` as specified in `dmoj/local_settings.py`.

```shell-session
(furaojsite) ./manage.py collectstatic
```

You will also need to generate internationalization files.

```shell-session
(furaojsite) ./manage.py compilemessages
(furaojsite) ./manage.py compilejsi18n
```

## Setting up Celery

The FuraOJ uses Celery workers to perform most of its heavy lifting, such as batch rescoring submissions. We will use Redis as its broker, though note that other brokers that Celery supports will work as well.

Start up the Redis server, which is needed by the Celery workers.

```shell-session
sudo systemctl start redis-server
```

Configure `local_settings.py` by uncommenting `CELERY_BROKER_URL` and `CELERY_RESULT_BACKEND`. By default, Redis listens on localhost port 6379, which is reflected in `local_settings.py`. You will need to update the addresses if you changed Redis's settings.

We will test that Celery works soon.

## Setting up database tables

We must generate the schema for the database, since it is currently empty.

```shell-session
(furaojsite) ./manage.py migrate
```

Next, load some initial data so that your install is not entirely blank.

```shell-session
(furaojsite) ./manage.py loaddata navbar language_small demo
```

!> Keep in mind that the `demo` fixture creates a superuser account with a username and password of `admin`. If your
site is exposed to others, you should change the user's password or remove the user entirely.

You should create an admin account with which to log in initially.

```shell-session
(furaojsite) ./manage.py createsuperuser
```

Now, you should verify that everything is going according to plan.

```shell-session
(furaojsite) ./manage.py check
```

## Setting up uWSGI

`runserver` is insecure and not meant for production workloads, and should not be used beyond testing.
In the rest of this guide, we will be installing `uwsgi` and `nginx` to serve the site, using `supervisord`
to keep `site` and `bridged` running. It's likely other configurations may work, but they are unsupported.

First, download our `uwsgi.ini` configuration file using `curl`. You should change the paths inside to reflect your install.

```shell-session
(furaojsite) curl -sSL -o uwsgi.ini https://raw.githubusercontent.com/furavietnam/furaoj-docs/refs/heads/main/sample_files/uwsgi.ini
```

If it says workers are spawned, it probably works.
You should Ctrl-C to exit.

## Setting up supervisord

You should now install `supervisord` and configure it.

```shell-session
(furaojsite) sudo apt install -y supervisor
```

Download `site.conf`, `bridged.conf`, and `celery.conf` directly to `/etc/supervisor/conf.d/` using `curl`:

```shell-session
(furaojsite) sudo curl -sS -o /etc/supervisor/conf.d/site.conf https://raw.githubusercontent.com/furavietnam/furaoj-docs/refs/heads/main/sample_files/site.conf
(furaojsite) sudo curl -sSL -o /etc/supervisor/conf.d/bridged.conf https://raw.githubusercontent.com/furavietnam/furaoj-docs/refs/heads/main/sample_files/bridged.conf
(furaojsite) sudo curl -sSL -o /etc/supervisor/conf.d/celery.conf https://raw.githubusercontent.com/furavietnam/furaoj-docs/refs/heads/main/sample_files/celery.conf
(furaojsite) sudo curl -sSL -o /etc/supervisor/conf.d/wsevent.conf https://raw.githubusercontent.com/furavietnam/furaoj-docs/refs/heads/main/sample_files/wsevent.conf
(furaojsite) sudo supervisorctl reread && sudo supervisorctl update
```

## Setting up nginx

Now, it's time to set up `nginx`.

```shell-session
(furaojsite) sudo apt install -y nginx
```

Download the sample `nginx.conf` via `curl` directly into `/etc/nginx/sites-available/furaoj`, then use `nano` to edit and configure it.

```shell-session
(furaojsite) sudo rm /etc/nginx/sites-enabled/default
(furaojsite) sudo curl -sSL -o /etc/nginx/sites-available/furaoj https://raw.githubusercontent.com/furavietnam/furaoj-docs/refs/heads/main/sample_files/nginx.conf
(furaojsite) sudo nano /etc/nginx/sites-available/furaoj
(furaojsite) sudo ln -sf /etc/nginx/sites-available/furaoj /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

You should be good to go. Visit the site at where you set it up to verify.

If it does not work, check `nginx` logs and `uwsgi` log `stdout`/`stderr` for details.

?> Now that your site is installed, remember to set `DEBUG` to `False` in `local_settings`. Leaving it `True` is a security risk.