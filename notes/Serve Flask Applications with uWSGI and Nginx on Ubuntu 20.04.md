# Run multiple flask applications on the same server using uWSGI and Nginx on Ubuntu 20.04

This manual guides you through the steps that are necessary to run multiple flask applications with uWSGI and Nginx on a Ubuntu 20.04 server that is hosted on linode.
For the server set up process, see the manual `Securing_linux_server.md`


Resources:

- [DigitalOcean Tutorial](https://www.digitalocean.com/community/tutorials/how-to-serve-flask-applications-with-uwsgi-and-nginx-on-ubuntu-20-04) by [Kathleen Juell](https://www.digitalocean.com/community/users/katjuell)
- [Why is WSGI necessary](https://www.fullstackpython.com/wsgi-servers.html)
- [Stackoverflow Post](https://stackoverflow.com/questions/48205495/python-how-to-run-multiple-flask-apps-from-same-client-machine)


## #1: Nginx

## Setting up an Nginx webserver 

#### Installation
```
sudo apt update
sudo apt install nginx
```

#### Configuration
Configure the uncomplicated firewall (ufw) to allow incoming nginx HTTP connections
```
sudo ufw allow 'Nginx HTTP'
```
The Nginx HTTP should now be listed as enabled applications: `sudo ufw app list`

Enable the system to start the nginx at next reboot, start nginx and check the web server's status:
```
sudo systemctl enable nginx
sudo systemctl start nginx

sudo systemctl status nginx
```

Nginx should now be running and accessible via http://<your_IP>

```bash
nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2021-04-30 09:36:55 UTC; 2s ago
       Docs: man:nginx(8)
    Process: 5779 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
    Process: 5780 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
   Main PID: 5781 (nginx)
      Tasks: 2 (limit: 1073)
     Memory: 1.6M
     CGroup: /system.slice/nginx.service
             ├─5781 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
             └─5782 nginx: worker process
```


## #2: Creating an example flask application

#### Installation
Install the required packages
```
sudo apt install python3-pip python3-dev python3-venv build-essential libssl-dev libffi-dev python3-setuptools
```

#### Create a new project folder and virtual environment
```
mkdir ~/flask_project
cd ~/flask_project
python3 -m venv flask_project_venv
source flask_project_venv/bin/activate
```

#### Create the example flask application
Wheel will ensure that the packages will install even if they are missing wheel archives.
```
pip install wheel
pip install flask
```

Create a sample app in the project folder `~/flask_project/flask_project.py `

```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "<h1 style='color:black'>Hello, World!</h1>"
if __name__ == "__main__":
    app.run(host='0.0.0.0')
```

By running this python script `python3 flask_project.py`, the just created website that shows "Hello, World!" should be accessible through <your_IP>:5000.

## #3: Configure uWSGI

#### Install uWSGI and enable the ufw firewall port 5000
```
pip install uwsgi
 
sudo ufw allow 5000
```

#### Create a WSGI Entry Point
This file `~/flask_project/wsgi.py` serves as an entry point and will tell the uWSGI server how to interact with it. It is located in the project folder.
```python
from flask_project import app

if __name__ == "__main__":
    app.run()
```

#### Testing uWSGI
```
uwsgi --socket 0.0.0.0:5000 --protocol=http -w wsgi:app
```
The website should now again be accessible through <your_IP>:5000.


#### Creat a uWSGI Config File
Deactivate the virtual environment and create a config file in the project folder called `flask_project.ini`.

`[uwsgi]` This header defines that this is a uWSGI config file.

`module = wsgi:app` specifies the module by referring to the `wsgi` entry point and the callable within this entry point file, which is in this case called `app`.

`master = true` defines that uwsgi will start up in master mode

`processes = 5` defines the number of worker processes to serve requests

`socket = flask_project.sock` refers to the unix socket that is used.

`chmod-socket = 660` defines the permission on the socket.

`vacuum = true` this will clean up the socket when the process stops

`die-on-term`  

```
[uwsgi]
module = wsgi:app

master = true
processes = 5

socket = flask_project.sock
chmod-socket = 660
vacuum = true

die-on-term = true
```

## #4: Creating a system Unit File
Systemd is a suite of tools that provides a fast and flexible init model for managing system services. Creating a systemd unit file will allow Ubuntu's init system to automatically start uWSGI and serve the Flask application whenever the server boots.

Create a unit file `/etc/systemd/system/flask_project.service`

```
[Unit]
Description=uWSGI instance to serve flask_project
After=network.target

[Service]
User=<your_user>
Group=www-data
WorkingDirectory=/home/<your_user>/flask_project
Environment="PATH=/home/<your_user>/flask_project/flask_project_venv/bin"
ExecStart=/home/<your_user>/flask_project/flask_project_venv/bin/uwsgi --ini flask_project.ini

[Install]
WantedBy=multi-user.target
```

Start the service and enable the service to start at boot

```
sudo systemctl start flask_project
sudo systemctl enable flask_project
```

The system should now be running without issues (`sudo systemctl status flask_project`)


## #5: Configuring Nginx to Proxy Requests
The up-running uWSGI application waits for request on the socket file in the project directory. Now, we will configure Nginx to pass web requests to that socket using the uwsgi protocol.

Create a new server block configuration in Nginx's `/etc/nginx/sites-available/flask_project` directory.

```
server {
    listen 80;
    server_name <your_IP>;

    location / {
        include uwsgi_params;
        uwsgi_pass unix:/home/<your_user>/flask_project/flask_project.sock;
    }
}
```

Link the file to the `sites-enabled` folder and unlink the default link

```
sudo ln -s /etc/nginx/sites-available/flask_project /etc/nginx/sites-enabled

sudo unlink /etc/nginx/sites-enabled/default
```

Test the nginx syntax
```
sudo nginx -t
```

Restart the Nginx to process the new configuration
```
sudo systemctl restart nginx
```

Adjust the firewall to allow access through port 5000
```
sudo ufw delete allow 5000
sudo ufw allow 'Nginx Full'
```

## #6: Debugging

#### Error logs:
- Nginx error logs: `sudo less /var/log/nginx/error.log`
- Nginx access logs: `sudo less /var/log/nginx/access.log`
- Nginx process logs: `sudo journalctl -u nginx`
- Flask app's logs: `sudo journalctl -u flask_project`