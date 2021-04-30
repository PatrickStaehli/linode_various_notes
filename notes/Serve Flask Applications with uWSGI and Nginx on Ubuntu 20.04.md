# Serve Flask Applications with uWSGI and Nginx on Ubuntu 20.04
## #1: Install Nginx on Ubuntu 20.04
```bash
sudo apt update
sudo apt install nginx
```
#### Install ufw firewall
```
sudo apt-get install ufw
sudo ufw allow ssh
sudo enable
```
List the enabled applications:
```
sudo ufw app list
```
Allow also 'Nginx HTTP'
```
sudo ufw allow 'Nginx HTTP'
```

#### Checking the Web Server's Status
```
systemctl status nginx
```

#### Re-enable the service to start up at boot:
```
sudo systemctl enable nginx
```

### Install Components
```
sudo apt install python3-pip python3-dev build-essential libssl-dev libffi-dev python3-setuptools
```

### Install Python Virtual Environment
```
sudo apt install python3-venv
mkdir ~/flask_project
cd ~/flask_project
python3 -m venc flask_project_venv
```
#### Activate environment
```
source flask_project_venv/bin/activate
```

### Setting up a Flask Application
Wheel will ensure that the packages will install even if they are missing wheel archives:
```
pip install wheel
```
### Install Flask and uWSGI
```
pip install uwsgi flask
```

## #2: Creating a Sample App
Create a sample app in the project folder ```vim ~/flask_project/flask_project.py ```

```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "<h1 style='color:blue'>Hello There!</h1>"
if __name__ == "__main__":
    app.run(host='0.0.0.0')
```

#### Enable UFW firewall port 5000
``` 
sudo ufw allow 5000
```

#### Creating a WSGI Entry Point
This file ```wsgi.py``` entry point will tell the uWSGI server how to interact with it.
```python
from flask_project import app

if __name__ == "__main__":
    app.run()
```

## #3: Configure uWSGI

#### Testing uWSGI
```
uwsgi --socket 0.0.0.0:5000 --protocol=http -w wsgi:app
```

#### Creat a uWSGI Config File
Deactivate the virtual environment and create a config file in the project folder called ```flask_project.ini ```
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

Create a unit file ```flask_project.service``` within the ```/etc/systemd/system```

```
[Unit]
Description=uWSGI instance to serve flask_project
After=network.target

[Service]
User=patrick
Group=www-data
WorkingDirectory=/home/patrick/flask_project
Environment="PATH=/home/patrick/flask_project/flask_project_venv/bin"
ExecStart=/home/patrick/flask_project/flask_project_venv/bin/uwsgi --ini flask_project.ini

[Install]
WantedBy=multi-user.target
```

## #5: Configuring Nginx to Proxy Requests
The uprunning uWSGI application waits for request on the socket file in the project directory. Now, we will configure Nginx to pass web requests to that socket using the uwsgi protocol.

Create a new server block configuration in Nginx's ```/etc/nginx/sites-available/flask_project``` directory.

```
server {
    listen 80;
    server_name YOUR-IP;

    location / {
        include uwsgi_params;
        uwsgi_pass unix:/home/patrick/flask_project/flask_project.sock;
    }
}
```

Link the file to the ```sites-enabled``` folder

```
sudo ln -s /etc/nginx/sites-available/flask_project /etc/nginx/sites-enabled
```

Unlink the default link
```
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

Adjust the user in ```/etc/nginx/nginx.conf```
```
user root;
```




