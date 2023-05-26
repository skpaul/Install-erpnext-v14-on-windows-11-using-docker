# Install ERPNext v14 Windows-11 Using Docker

An easy and step-by-step guide to install `ERPNext` application in Windows 11 using Docker.

### Pre-requisites 

      Docker Desktop
      git
      Wnidows 11 
      VS Code


### Check Docker version
```shell
docker version
git version
```

### Create a directory in your preferred location

```shell
mkdir mysite #assuming that your project name is mysite
cd mysite
```

### Clone frappe_docker 

```sh
git clone https://github.com/frappe/frappe_docker.git .
```

### Copy `devcontainer-example`  to `.devcontainer` folder

```shell
ren devcontainer-example .devcontainer
```

### Customize `docker-compose.yml`

- Rename `.devcontainer/docker-compose.yml` as `.devcontainer/docker-compose (MAIN).yml`
- Download `docker-compose.yml` from here.
- Replace `.devcontainer/docker-compose.yml` by the downloaded file.

### Configure .vscode 

Copy `development/vscode-example` to `development/.vscode`. This will setup basic configuration for debugging. 

```sh
cd development
ren vscode-example .vscode
cd ..   #go back to previous dir
```

### Install VSCode Remote Containers extension

- Open VS Code
- Install `Dev Containers` extension

### Open  `mysite`  folder in VS Code

After the extensions are installed successfully-

```shell
code .
```

### Execute  `Reopen in Container`  command in VS Code

- Press `Ctrl + Shift + P`
- Select  `Remote-Containers: Reopen in Container`

*You can also click in the bottom left corner to access the remote container menu.*

### Initialize  `frappe bench`

Run the following commands in the terminal inside the container. 
*You might need to create a new terminal in VSCode.*


```shell
bench init --skip-redis-config-generation --frappe-branch version-14 frappe-bench
```

```shell
cd frappe-bench 
```

### Setup hosts

We need to tell bench to use the right containers instead of localhost. Run the following commands inside the container:

```sh
bench set-config -g db_host mariadb
```
```shell
bench set-config -g redis_cache redis://redis-cache:6379
```

```shell
bench set-config -g redis_queue redis://redis-queue:6379
```

```shell
bench set-config -g redis_socketio redis://redis-socketio:6379
```

For any reason the above commands fail, set the values in `common_site_config.json` manually.

```json
{
  "db_host": "mariadb",
  "redis_cache": "redis://redis-cache:6379",
  "redis_queue": "redis://redis-queue:6379",
  "redis_socketio": "redis://redis-socketio:6379"
}
```

### Create a new site
Sitename **MUST** end with `.localhost` for trying deployments locally. 
#### Options

```shell
--db-name          # Set the Database name for new site
--db-password      # Set the Database password for new site
--db-type          # Select the Database Type for new site- "postgres" or "mariadb". Default is "mariadb"
--db-host          # Set Database Host for new site
--db-port          # Set Database Port for new site
--db-root-username # Specify Root username for MariaDB or Postgres
--db-root-password # Specify Root password for MariaDB or Postgres
--admin-password   # Specify the Administrator password for new site
--source_sql       # Initiate database with a SQL file
--install-app      # Install app after installation
```



```shell
# The default mariadb root password is '123'. Must Use the same.
bench new-site mysite.localhost --admin-password 654321 --db-type mariadb --db-root-username root --mariadb-root-password 123  --db-name mysite --db-password 321 --no-mariadb-socket
```


This will create a new site and a `mysite.localhost` directory under `frappe-bench/sites`.
The option `--no-mariadb-socket` will configure site's database credentials to work with docker.
You may need to configure your system /etc/hosts if you're on Linux, Mac, or its Windows equivalent.

### Set bench developer mode on the new site

```shell
bench --site mysite.localhost set-config developer_mode 1
bench --site mysite.localhost clear-cache   
```

### Install ERPNext

```shell
# we are using v.14
bench get-app --branch version-14 --resolve-deps erpnext
bench --site mysite.localhost install-app erpnext
```

### Start Frappe bench 

```shell
bench start
```

Wait 20-30 seconds to complete the startup process.

You can now login with user **Administrator** and the password you choose when creating the site. Your website will now be accessible at location `http://mysite.localhost:8000` 

## Start Frappe with Visual Studio Code Python Debugging

To enable Python debugging inside Visual Studio Code, you must first install the `ms-python.python` extension inside the container. This should have already happened automatically, but depending on your VSCode config, you can force it by:

- Click on the extension icon inside VSCode
- Search `ms-python.python`
- Click on `Install on Dev Container: Frappe Bench`
- Click on 'Reload'

We need to start bench separately through the VSCode debugger. For this reason, **instead** of running `bench start` you should run the following command inside the frappe-bench directory:

```shell
honcho start \
    socketio \
    watch \
    schedule \
    worker_short \
    worker_long \
    worker_default
```

Alternatively you can use the VSCode launch configuration "Honcho SocketIO Watch Schedule Worker" which launches the same command as above.

This command starts all processes with the exception of Redis (which is already running in separate container) and the `web` process. The latter can can finally be started from the debugger tab of VSCode by clicking on the "play" button.

You can now login with user `Administrator` and the password you choose when creating the site, if you followed this guide's unattended install that password is going to be `admin`.

To debug workers, skip starting worker with honcho and start it with VSCode debugger.

For advance vscode configuration in the devcontainer, change the config files in `development/.vscode`.

## Developing using the interactive console

You can launch a simple interactive shell console in the terminal with:

```shell
bench --site mysite.localhost console
```

More likely, you may want to launch VSCode interactive console based on Jupyter kernel.

Launch VSCode command palette (cmd+shift+p or ctrl+shift+p), run the command `Python: Select interpreter to start Jupyter server` and select `/workspace/development/frappe-bench/env/bin/python`.

The first step is installing and updating the required software. Usually the frappe framework may require an older version of Jupyter, while VSCode likes to move fast, this can [cause issues](https://github.com/jupyter/jupyter_console/issues/158). For this reason we need to run the following command.

```shell
/workspace/development/frappe-bench/env/bin/python -m pip install --upgrade jupyter ipykernel ipython
```

Then, run the command `Python: Show Python interactive window` from the VSCode command palette.

Replace `mysite.localhost` with your site and run the following code in a Jupyter cell:

```python
import frappe

frappe.init(site='mysite.localhost', sites_path='/workspace/development/frappe-bench/sites')
frappe.connect()
frappe.local.lang = frappe.db.get_default('lang')
frappe.db.connect()
```

The first command can take a few seconds to be executed, this is to be expected.

## Manually start containers

In case you don't use VSCode, you may start the containers manually with the following command:

### Running the containers

```shell
docker-compose -f .devcontainer/docker-compose.yml up -d
```

And enter the interactive shell for the development container with the following command:

```shell
docker exec -e "TERM=xterm-256color" -w /workspace/development -it devcontainer-frappe-1 bash
```

## Connect from PHPMyAdmin/HeidiSQL

```
Host: localhost
Port: 3306
User Name: root
Password: 123
```

