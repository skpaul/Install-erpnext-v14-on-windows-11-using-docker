# Install ERPNext v14 Windows-11 Using Docker

An easy and step-by-step guide to install Frappe Bench in Windows 11 using Docker and install Frappe/ERPNext application

### Pre-requisites 

      Docker Desktop
      git
      Wnidows 11 
      VS Code


### Check Docker version
    docker version
    git version

### Clone frappe_docker and move to frappe_docker folder

```sh
git clone https://github.com/frappe/frappe_docker.git
cd frappe_docker
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
ren development/vscode-example development/.vscode
```

### Install VSCode Remote Containers extension

```shell
Open vscode and install 'Dev Containers' extension
```

### Open  `frappe_docker`  folder in VS Code

After the extensions are installed successfully-

```shell
code .
```

### Execute  `Reopen in Container`  command in VS Code

- Press `Ctrl + Shift + P`
- Select  `Remote-Containers: Reopen in Container`

*You can also click in the bottom left corner to access the remote   container menu.*

### Initialize  `frappe bench`

Run the following commands in the terminal inside the container. You might need to create a new terminal in VSCode.


```shell
bench init --skip-redis-config-generation --frappe-branch version-14 frappe-bench
cd frappe-bench 
```

### Setup hosts

We need to tell bench to use the right containers instead of localhost. Run the following commands inside the container:

```sh
bench set-config -g db_host mariadb
bench set-config -g redis_cache redis://redis-cache:6379
bench set-config -g redis_queue redis://redis-queue:6379
bench set-config -g redis_socketio redis://redis-socketio:6379
```
  For any reason the above commands fail, set the values in common_site_config.json manually.

```json
{
  "db_host": "mariadb",
  "redis_cache": "redis://redis-cache:6379",
  "redis_queue": "redis://redis-queue:6379",
  "redis_socketio": "redis://redis-socketio:6379"
}
```

### Create a new site
Sitename **MUST** end with `.localhost` for trying deployments locally. MariaDB root password: 123
```shell
bench new-site mysite.localhost --no-mariadb-socket    
```

The same command can be run non-interactively as well:

```shell
bench new-site mysite.localhost --mariadb-root-password 123456 --admin-password 123456 --no-mariadb-socket
```

The command will ask the MariaDB root password. The default root password is `123`.
This will create a new site and a `mysite.localhost` directory under `frappe-bench/sites`.
The option `--no-mariadb-socket` will configure site's database credentials to work with docker.
You may need to configure your system /etc/hosts if you're on Linux, Mac, or its Windows equivalent.

NOTE

To setup site with PostgreSQL as database use option `--db-type postgres` and `--db-host postgresql`. 

```shell
bench new-site mysite.localhost --db-type postgres --db-host postgresql
```



### STEP 9 Set bench developer mode on the new site

```shell
bench --site mysite.localhost set-config developer_mode 1
bench --site mysite.localhost clear-cache   
```


​    
### STEP 10 Install ERPNext

```shell
# --branch is optional
bench get-app --branch version-14 --resolve-deps erpnext
bench --site mysite.localhost install-app erpnext
```


​    
​    

### STEP 11 Start Frappe bench 

```shell
bench start
```

You can now login with user Administrator and the password you choose when creating the site. Your website will now be accessible at location `http://mysite.localhost:8000`

​    

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

## 