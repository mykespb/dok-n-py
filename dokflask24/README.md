Docker and Flask and Nginx test, based on internet exersize book
=================================================================

Date: 2019-09-07
Version: 1.0
Experimenter: Mikhail Kolodin


Initial version
----------------------

Building a Flask app with Docker | Learning Flask Ep. 24
Containerizing a Flask application with Docker, Nginx & uWSGI using Docker Compose

Julian Nash pythonise.com
Julian Nash
27 Mar 2019

https://pythonise.com/feed/flask/building-a-flask-app-with-docker-compose

In this guide, we're going to build a simple Flask application using Docker, more specifically with Docker Compose. A powerful and convenient way to work with and configure Docker containers & services.

We'll be using Nginx as our HTTP webserver and uWSGI as our application server, just like in the previous guide to deploying a Flask app on a virtual machine.

We're going to cover everything from scratch so don't worry if you've never worked with Docker or Docker Compose before, just make sure you've got them both installed on your machine and we'll try our best to cover the concepts as we go.

To install Docker, head to the link below:

Install Docker CE
To install Docker compose, head to the link below:

Install Docker Compose
Armed and ready with Docker and Docker Compose? Let's get started!

Application structure
Here's an overview of how our application is going to look:

app
├── docker-compose.yml
├── flask
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── app
│   │   ├── __init__.py
│   │   └── views.py
│   ├── app.ini
│   ├── requirements.txt
│   └── run.py
├── nginx
│   ├── Dockerfile
│   └── nginx.conf
├── .gitignore
└── readme.md
We'll get started by creating a new directory for our new project and move into it. We'll call ours app but feel free to name it whatever you like:

mkdir app && cd app
Inside of our app parent directory, we're going to create a couple more directorires, one for each container - flask and nginx:

mkdir flask nginx
While we're here, let's create our docker-compose.yml file here in the app directory. We'll use this file to define our application services shortly:

touch docker-compose.yml
Feel free to create a .gitignore and readme.md too if you're going to be pushing this project up to Github!

touch .gitignore readme.md
Your project should now look something like this:

app
├── docker-compose.yml
├── flask/
├── nginx/
├── .gitignore
└── readme.md
Let's start off by building the basic Flask app and testing it locally before we do anything else.

The Flask app
Move into the flask directory:

cd flask
We'll start by creating a typical Flask project structure that you're likely to be familiar with, but first, let's create a virtual environment:

python -m venv env
Activate it:

source env/bin/activate
The only dependencies we require are flask and uwsgi, let's install them with pip:

pip install flask uwsgi
Once the packages are installed, we'll generate a requirements.txt:

pip freeze > requirements.txt
Now we can move on to building out out app!

Go ahead and create a file called run.py. This will be the entrypoint to our app:

touch run.py
Open up run.py and add the following:

run.py

from app import app

if __name__ == "__main__":
    app.run()
We also need an ini file for our uWSGI configuration, go ahead and create app.ini and add the following:

app.ini

[uwsgi]
wsgi-file = run.py
callable = app
socket = :8080
processes = 4
threads = 2
master = true
chmod-socket = 660
vacuum = true
die-on-term = true
Let's quickly touch on a few of the settings in app.ini:

wsgi-file = run.py - The file containing the callable (app)
callable = app - The callable object itself
socket = :8080 - The socket uwsgi will listen on (More on that later!)
You can read more about the uwsgi config here

We need a directory for our application module to live, go ahead and create a new directory called app and move into it:

mkdir app && cd app
We need an __init__.py file. Create it, open it up and add the following:

app/__init__.py

from flask import Flask

app = Flask(__name__)

from app import views
__init__.py simply imports Flask, creates the app object and imports our views.py file.

The last file we need here is views.py containing our app routes. Go ahead and create it and add the following:

app/views.py

from app import app
import os

@app.route("/")
def index():

    # Use os.getenv("key") to get environment variables
    app_name = os.getenv("APP_NAME")

    if app_name:
        return f"Hello from {app_name} running in a Docker container behind Nginx!"

    return "Hello from Flask"
It's a simple route with the addition of app_name = os.getenv("APP_NAME"), which will become apparent later when we use docker-compose.yml to set some enviromment variables.

Your flask directory should now look like this:

flask
├── app
│   ├── __init__.py
│   └── views.py
├── app.ini
├── requirements.txt
└── run.py
At this point, we're ready to test our application locally!

Testing the Flask app locally
Move back up to the flask directory:

cd ..
Just like when we normally run a Flask app, we need to set a few environment variables. Make sure you're in the flask directory and run the following:

export FLASK_APP=run.py
export FLASK_ENV=development
Now, run the app:

flask run
Head to http://127.0.0.1:5000/ in your browser and you should see:

Hello from Flask
If everything worked, use Ctrl + c to kill the Flask development server.

Flask Dockerfile
A Dockerfile is a special type of text file that Docker will use to build our containers, following a set of instruction that we provide.

We need to create a Dockerfile for every image we're going to build. In this case, both one for Flask and one for Nginx.

Make sure you're in the flask directory, create a Dockerfile:

touch Dockerfile
Add the following:

flask/Dockerfile

# Use the Python3.7.2 image
FROM python:3.7.2-stretch

# Set the working directory to /app
WORKDIR /app

# Copy the current directory contents into the container at /app 
ADD . /app

# Install the dependencies
RUN pip install -r requirements.txt

# run the command to start uWSGI
CMD ["uwsgi", "app.ini"]
Let's touch on the contents of our Dockerfile

FROM python:3.7.2-stretch - Pull the python:3.7.2-stretch container base image
WORKDIR /app - Sets our working directory as /app
ADD . /app - Copies everything in the current directory to /app in our container
RUN pip install -r requirements.txt - Installs the packages from requirements.txt
CMD ["uwsgi", "app.ini"] - Starts the uwsgi server
You can read the Dockerfile reference here

At this point, your flask directory should look like this:

flask
├── Dockerfile
├── app
│   ├── __init__.py
│   └── views.py
├── app.ini
├── requirements.txt
└── run.py
The ADD . /app instruction in our Dockerfile is going to copy everything in the flask directory into the new container, but we don't need it to copy our virtual environment or any cached Python files.

To prevent this, we can create a .dockerignore file, just like how you'd create a .gitignore file , prioviding a list of file or directory names that we don't want docker to copy over2 to our image.

Go ahead and create it:

touch .dockerignore
Add the following:

env/
__pycache__/
At this point, your finished flask directory should look like this:

flask
├── Dockerfile
├── .dockerignore
├── app
│   ├── __init__.py
│   └── views.py
├── app.ini
├── requirements.txt
└── run.py
We're going to use the docker-compose.yml file to instruct Docker to build this image shortly, however we still need to setup Nginx.

Nginx
Go ahead and move into the nginx directory we created in the base of our application:

cd ../nginx/
We'll start by creating an Nginx config file that we'll use to setup a server block and route traffic to our application.

Go ahead and create a new file called nginx.conf:

touch nginx.conf
Open it up and add the following:

nginx/nginx.conf

server {

    listen 80;

    location / {
        include uwsgi_params;
        uwsgi_pass flask:8080;
    }

}
Inside of the server block:

listen 80; - Instructs Nginx to listen for requests on port 80 (HTTP)
include uwsgi_params; - Incluses the uwsgi_params file
To map the server block to an IP address, you'd set the server_name value to your servers' IP and place it just under listen 80;:

server_name  1.2.3.4;
To map the server block to a domain:

server_name example.com www.example.com;
uwsgi_pass flask:8080; is where things get a little interesting!

In our last Flask guide, we deployed a Flask app to a virtual machine, also with Nginx and uWSGI and our Nginx server block looked like this:

server {

    listen 80;

    location / {
        include uwsgi_params;
        uwsgi_pass unix:/home/username/app/app.sock;
    }

}
You'll notice the uwsgi_pass value is differet between the two.

On the virtual machine (I.e not using Docker containers) the value for uwsgi_pass is unix:/home/username/app/app.sock, a full path to a local socket file.

In this case, HTTP requests coming in are handled by Nginx and proxied to unix:/home/username/app/app.sock; where a uwsgi application server is listening and waiting to handle and requests.

However, in the case of our Docker example, we have both Flask and Nginx each in their own container. So how do they communicate?

Docker Compose sets up a single network for our application, where each container is freely reachable by any other container on the same network, and discoverable by their container name.

Naming our Flask container as flask will give it a hostname of flask. Naming our Nginx container nginx will give it that hostname and so on..

We configured uwsgi to listen on socket :8080 in the app.ini file, so now any HTTP requests received by Nginx are proxied to the Flask container using uwsgi_pass flask:8080;.

We name our containers in docker-compose.yml so we'll touch more on that shortly.

On a side note - we'll cover how to setup a custom domain and HTTPS using certbot in a future guide.

You can learn more about networking in Compose here

To finish up our basic Nginx container, we need to create a Dockerfile to build the image.

Nginx Dockerfile
Make sure you're in the nginx directory and create a Dockerfile:

touch Dockerfile
Open it up and add the following:

nginx/Dockerfile

# Use the Nginx image
FROM nginx

# Remove the default nginx.conf
RUN rm /etc/nginx/conf.d/default.conf

# Replace with our own nginx.conf
COPY nginx.conf /etc/nginx/conf.d/
We've only got 3 instructions in our Nginx Dockerfile:

FROM nginx - Pulls the oficial Nginx container image
RUN rm /etc/nginx/conf.d/default.conf - Removes the default Nginx config file
COPY nginx.conf /etc/nginx/conf.d/ - Copies the nginx.conf file we just created into the container
At this point, your nginx directory should look like this:

nginx
├── Dockerfile
└── nginx.conf
Lest thing to do now is create our docker-compose.yml.

Docker compose
Docker Compose is a powerful tool with lots of features and configuration options available, allowing us to define, build and configure our services & containers all from one place.

We're only touching on some of the basics in this guide but we'd definitely recommend having a read through the documentation here to get a better understanding of the features and options of Compose.

Open up the docker-compose.yml file you created earlier and add the following:

docker-compose.yml

version: "3.7"

services:

  flask:
    build: ./flask
    container_name: flask
    restart: always
    environment:
      - APP_NAME=MyFlaskApp
    expose:
      - 8080

  nginx:
    build: ./nginx
    container_name: nginx
    restart: always
    ports:
      - "80:80"
Let's go through what we've got here and briefly touch on some of the concepts:

Every docker-compose.yml file must start with the version, followed by the services we want to build using services: and indenting any individual services thereafter. In our case - version: "3.7".

We've defined 2 individual services in the services block and named them flask and nginx respectively.

build: ./flask - Instructs Docker to build the image using the Dockerfile found in the flask directory (relative to the docker-compose.yml file)
container_name: flask - Gives our container the name of flask, also assigning it that hostname as we mentioned earlier
restart: always - Makes the container always restart
environment - A place for us to define environment variables for the container.
expose - Exposes internal ports to other containers and services on the same network
You'll see we have a similar setup with the nginx service with the only difference being ports instead of expose, the two of which serve a fundamentally diffefrent purpose.

ports are mapped as HOST:CONTAINER and will expose ports to the outside world, which in our case has mapped port 80 on the host machine to port 80 of our Nginx container.

ports:
  - "80:80"
expose on the other hand has simply opened up port 8080 internally, allowing our Nginx container to communicate with the uwsgi server inside of the flask container listening on socket 8080!

expose:
  - 8080
Right now, your project should be looking like so:

app
├── docker-compose.yml
├── flask
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── app
│   │   ├── __init__.py
│   │   └── views.py
│   ├── app.ini
│   ├── requirements.txt
│   └── run.py
├── nginx
│   ├── Dockerfile
│   └── nginx.conf
├── .gitignore
└── readme.md
Save the file and jump back into your terminal. We're going to build and test out our app.

Building & testing
Run the following command in the same directory as docker-compose.yml to build the services:

docker-compose build
And now run the following to create and start the containers:

docker-compose up
You can optionally run the following to achieve the same result:

docker-compose up --build
You'll notice we didn't have to pass a filename to the command. Compose will look in the current directory for a docker-compose.yml file to build!

Open up http://127.0.0.1/ or http://localhost/ to see your Flask app in action. You should see:

Hello from MyFlaskApp running in a Docker container behind Nginx!
To stop the service, hit Ctrl + c.

Docker compose commands
docker-compose comes with quite a large number of commands and options which can be found by running:

docker-compose
We're not going to cover many ot the commands in this guide, so be sure to read the documentation or at least run docker-compose and have a read.

Run the following to list any running images:

docker-compose images
Container    Repository     Tag       Image Id      Size
---------------------------------------------------------
flask       docker_flask   latest   d7ea63adcedf   899 MB
nginx       docker_nginx   latest   8b3031c24599   104 MB
docker-compose ps will list any running containers:

docker-compose ps
Name          Command          State         Ports
---------------------------------------------------------
flask   uwsgi app.ini          Up      8080/tcp
nginx   nginx -g daemon off;   Up      0.0.0.0:80->80/tcp
To stop our services:

docker-compose stop
Making changes
If you make any changes to the application, you'll have to rebuid the images by running:

docker-compose up --build
The best way to build and test your app is to use the virtual environment we created earlier and the Flask development server.

Once you're happy with how your application is running on the development server, you can test it locally by building the services with Docker Compose.

Next steps
This guide was designed as a gentle introduction to building a basic flask application and running it begind Nginx and uWSGI using containers and Docker Compose, along with some of the Docker and Compose concepts.

Practically speaking, most applications are likely to have more services running alongside them, such as a database, cache, task queue etc, in addition to a domain name, certificates and served over HTTPS.

Some services are also likely to require persistent data, which can be achieved by configuring volumes for things such as database data, logs or any other data we'd like to make available between containers.

Any data stored in a container will be destroyed when we rebuild it, however we can setup a volume on the host machine as a safe and persistent place to store it. When the containers are destroyed and rebuilt, our data remains!

The great thing about Docker is that now we can simply bolt on additional services, build new images and bring them together using Docker Compose.

In a future part of this series we'll be adding additional services, including:

Setting up a MongoDB database container
Setting up Redis container as a cache
Setting up Certbot to generate a self signed certificate
Deploying the application
Thanks for reading!

