Introduction to uWSGI | Learning Flask Ep. 25 - An introduction to running Flask (and other Python applications) with uWSGI
=================================================================

Date: 2019-09-11
Version: 0.1
Experimenter: Mikhail Kolodin


0. Initial setup.
...
TBC



Experimental version
----------------------

Run 
docker-compose up
and see http://localhost:8088

It uses venv, so start it when needed for development.

It runs python 3.7.3 inside.

1.2. Improved version of packages installation and updating order.


Initial version
----------------------

Introduction to uWSGI | Learning Flask Ep. 25
An introduction to running Flask (and other Python applications) with uWSGI
Julian Nash pythonise.com
Julian Nash
09 Apr 2019

https://pythonise.com/feed/flask/python-flask-uwsgi-introduction

The uwsgi protocol is the native protocol used by the uWSGI server and is a popular, solid and reliable application server (And web server!) for many Python web applications, commonly coupled with Flask, Django and other web servers such as Nginx or Apache.

uWSGI itself is written in C, however has libraries/plugins available for many programming languages including Python, Ruby & more. In this guide we'll be using the python uwsgi package available via a simple pip install uwsgi command.

In this short guide, I'll give you a quick start introduction to using uWSGI to run your Flask applications on a couple of different setups including:

Using uWSGI as a stand alone web server
uWSGI as a Flask application server, with Nginx as the web server
uWSGI and Nginx serving a Flask application using Docker

What is uWSGI
From the uWSGI documentation:

"A web server faces the outside world. It can serve files (HTML, images, CSS, etc) directly from the file system. However, it can’t talk directly to Python applications; it needs something that will run the application, feed it requests from web clients (such as browsers) and return responses.

A Web Server Gateway Interface - WSGI - does this job. WSGI is a Python standard.

uWSGI is a WSGI implementation."

The concept
Web client (Browser/application) <-> Web server (Nginx/Apache) <-> Socket <-> uwsgi <-> Python application (Flask/Django)
uWSGI as a standalone HTTP webserver
Whilst you'll most commonly see uWSGI used behind a web server such as Nginx or Apache, uWSGI can also be used as an HTTP/HTTPS router, proxy and load balancer.

To get started, we'll create a very simple "Hello world" Flask application and run it locally using uWSGI.

In a new directory, we'll create a virtual environment with the following:

python -m venv env
Activate the virtual environment (use set instead of source if on Windows):

source env/bin/activate
Installing uWSGI
uwsgi can be installed with a simple pip install uwsgi.

We're going to build a simple Flask app and only need 2 dependencies, flask and uwsgi:

pip install flask uwsgi
A simple Flask app
Create a new file called run.py and add the following:

from flask import Flask

app = Flask(__name__)

@app.route("/")
def index():
    return "Hello world"

if __name__ == "__main__":
    app.run()
You'll notice the inclusion of the if __name__ == "__main__": block as you'll likely want to develop your Flask applications using the built in Flask development server. This part isn't actually required to run the application with uWSGI.

Running with uWSGI
To run the application with uWSGI using a very basic configuration:

uwsgi --http 127.0.0.1:8080 --wsgi-file run.py --callable app
You'll see some output in the terminal:

*** Starting uWSGI 2.0.18 (64bit) on [Mon Apr  8 23:31:30 2019] ***
compiled with version: 7.3.0 on 05 April 2019 11:18:20
os: Linux-4.4.0-17763-Microsoft #379-Microsoft Wed Mar 06 19:16:00 PST 2019
nodename: jnwt
machine: x86_64
clock source: unix
detected number of CPU cores: 24
# etc...
You'll also see some output on each request:

[pid: 659|app: 0|req: 1/1] 127.0.0.1 () {42 vars in 1090 bytes} [Tue
Apr  9 00:02:21 2019] GET / => generated 11 bytes in 0 msecs (HTTP/1.1 200) 2 headers in 79 bytes (1 switches on core 0)
Head to http://127.0.0.1:8080/ in a browser window to see your Flask application being served by uWSGI.

To stop uWSGI running, just hit Ctrl + c.

What have we just done?
We've told uWSGI to run an HTTP web server on port 8080 using the callable object app in the file run.py.

uwsgi - Starts the uwsgi binary found in the newly created virtual environment @ env/bin/uwsgi
--http 127.0.0.1:8080 - Tells uWSGI to listen on port 8080 for requests. The http option makes uWSGI natively speak HTTP
--wsgi-file run.py - Tells uWSGI where to find the callable object. In our case it's the app object we created in run.py
--callable app - Tells uWSGI the name of our callable object, again it's the app object we created with app = Flask(__name__)
Whilst it's perfectly fine to start the uWSGI server from the commend line, there's a better way!

Rather than passing arguments and options to the uwsgi command, we can put our configuration in an ini file.

uWSGI configuration file
Rather than passing arguments to uwsgi using the command line, we can put them in an ini file for convenience.

For this example, we'll create a new file called http.ini and add our options as before:

[uwsgi]
http = :8080
wsgi-file = run.py
callable = app
We start by creating a [uwsgi] section, followed by the same options and values as before, only separated with an equals - just like assigning a value to a Python variable.

We'll return to this file and add some more options shortly. For now, let's run our app using this configuration file.

Running with an ini file
To run our application with uWSGI using a ini configuration file, simply call the uwsgi command followed by the name or path to the config file:

uwsgi http.ini
As before, you'll see some information in the terminal, letting you know uWSGI has started.

Head to http://127.0.0.1:8080/ in a browser window to see your app being served.

Let's dig a little deeper into the uWSGI configuration options.

uWSGI options
uWSGI comes with heaps of configuration options to cover a wide range of use cases.

In this guide, we'll take a look at some of the common configuration options for use with Python applications like Flask and Django.

Open up http.ini and add the following additions, we'll discuss them after:

master = true
processes = 4
threads = 8
master = true - Respawns processes when they die and handles much of the built-in prefork+threading multi-worker management, generally always advised
processes = n - Adds concurrency to the app by adding n additional processes
threads = n - Adds additional cuncurrency by adding n additional threads
Note - Unfortunately, there's no magic formula for specifying the numbers of processes and threads, so you'll have to do some trial and error to find out what gives you the best results.

As a starting point, I tend to set processes to the number of CPU cores on the mahcine and threads at 2 per CPU core.

A useful tool for monitoring your application and resources is uwsgitop which we'll touch on later.

If you run the application again with:

uwsgi http.ini
You'll notice some additional output in the terminal, followed by the request information as before:

python threads support enabled
# ...
*** Operational MODE: preforking+threaded ***
# ...
*** uWSGI is running in multiple interpreter mode ***
spawned uWSGI master process (pid: 674)
spawned uWSGI worker 1 (pid: 675, cores: 8)
spawned uWSGI worker 2 (pid: 683, cores: 8)
spawned uWSGI worker 3 (pid: 691, cores: 8)
spawned uWSGI worker 4 (pid: 699, cores: 8)
spawned uWSGI http 1 (pid: 707)
# ...
[pid: 691|app: 0|req: 1/2] 127.0.0.1 () {42 vars in 1090 bytes} [Tue
Apr  9 01:28:03 2019] GET / => generated 11 bytes in 2 msecs (HTTP/1.1 200) 2 headers in 79 bytes (1 switches on core 0)
Our 4 processes have been spawned and uWSGI will balance requests across them. Pretty cool!

Let's add some more options to our ini file so we can monitor the application more closely:

memory-report = true
stats = stats.sock
memory-report = true - This option will include memory usage information in every request
stats = stats.sock - Will create a socket file called stats.sock which we can pass to uwsgitop for live monitoring
Saving and running the application will produce the following output in the terminal on every request:

{address space usage: 771027046400 bytes/735308MB} {rss usage: 5484544 bytes/5MB} [pid: 884|app: 0|req: 3/8] 127.0.0.1 () {42 vars in 1090 bytes} [Tue Apr  9 11:30:30 2019] GET / => generated 11 bytes in 0 msecs (HTTP/1.1 200) 2 headers in 79 bytes (1 switches on core 5)
You'll see we now get some memory information printed to the terminal every time a request is made to the app.

Monitoring with uwsgitop
To use uwsgitop, install it with:

pip install uwsgitop
With uwsgi running the app, open up a new terminal (be sure to activate the virtual environment) and start uwsgitop with:

uwsgitop stats.sock
You'll see something very similar to the following:

WID    %       PID     REQ     RPS     EXC     SIG     STATUS  AVG     RSS     VSZ     TX      ReSpwn  HC      RunT            LastSpwn
3      36.8    943     25934   53      0       0       idle    0ms     11.6M   1010.3G 2.2M    1       0       10532.494       11:51:48
2      28.4    935     19994   24      0       0       idle    0ms     11.6M   1010.3G 1.7M    1       0       7756.532        11:51:48
1      22.5    927     15830   41      0       0       idle    0ms     10.3M   1010.3G 1.4M    1       0       6033.569        11:51:48
4      12.4    951     8729    22      0       0       idle    0ms     6.6M    1010.3G 767.2K  1       0       3467.54         11:51:48
See the table below for what these stats mean:

Field	Description
WID	The worker ID
%	The worker usage
PID	The worker PID (process identifier)
REQ	The number of requests the worker has executed since last respawn
RPS	Requests per second
EXC	Exceptions
SIG	Number of managed uwsgi signals
STATUS	If the worker is busy or free
AVG	The average request time
RSS	The worker RSS (Resident set size - Linux memory management)
VSZ	The worker VSZ (Virtual Memory Size - Linux memory management)
TX	How much data was transmitted by the worker
RunT	How long the worker has been running
You can read more about uwsgitop here

Stress testing with Locust
To really see uwsgitop in action, we're going to stress test our app with Locust.

Locust is a fantastic Python package for stress testing applications, allowing us to simulate thousands of users making concurrent requests to our app.

To get started, we need to install it with pip:

pip install locustio
To start stress testing out app, we need to create a Locustfile. Go ahead and create a file called locustfile.py and add the following:

from locust import HttpLocust, TaskSet, task

class UserBehavior(TaskSet):

    @task(1)
    def index(self):
        self.client.get("/")

class WebsiteUser(HttpLocust):
    task_set = UserBehavior
    min_wait = 5000
    max_wait = 9000
Locust is a big and powerful package and we'll cover it in more detail in a separate guide. You can read more about it here.

For now, we're just going to make GET requests to our one and only route.

Make sure your app is running with uwsgi http.ini and you have uwsgitop stats.sock running in a separate terminal.

In a third terminal, activate your virtual environment and start Locust by running:

locust --host=http://127.0.0.1:8080
We pass --host=http://127.0.0.1:8080 to the locust command as that's the port our Flask app is running on.

You should see some output similar to the following:

[2019-04-09 12:21:35,910] jnwt/INFO/locust.main: Starting web monitor
at *:8089
Locust runs its own local web server and web application, which we'll use to setup, start and manage our stress test. In our case, it's running on port 8089 as shown by the output above.

Go to http://127.0.0.1:8089/ in a new browser window (or whatever port number Locust tells you it's running on) to get started.

Starting a Locust swarm
You'll be greeted with a start page where you'll be asked for the number of users to simulate and a hatch rate.

Locust Web UI

Go ahead and enter some numbers, a good place to start is 1000 for number of users and 10 for hatch rate.

Click on "Start swarming"

You'll be taken to a page displaying information about the swarm, feel free to take a look around and explore the Locust UI.

Over in the terminal running uwsgitop, you'll see uWSGI in action.

You can stop Locust from the web UI by clicking the big red stop button in the top right corner, or Ctrl + c in the terminal.

We'll leave you to explore and play with Locust as we'll be covering it in a separate article.

Let's get back to uWSGI.

More uWSGI configuration
uWSGI strongly encourages changing the user and group privelages (more specifically, dropping root privelages), achieved by setting the uid and gid options with a username and group respectively.

This will depend on your application setup, for example...

Running uWSGI as a standalone HTTP webserver, without Nginx or Apache as a reverse proxy; assign uid & gid as a non root user or nobody.

uid = username
gid = group

; Or use the nobody user and group
uid = nobody
gid = nobody
I'm not sure if setting uid and gid as nobody effects uWSGI's logging, I still need to test this.

Running uWSGI as an application server, behind Nginx or Apache; you may want to set these values as the same user/group as the web server, such as www-data (They may have different user/group names on other systems such as data or web)

uid = www-data
gid = www-data
The www-data group typically has zero privellages and cannot write to any file on the entire file system, however can write to some specific files.

Running uWSGI & Flask in a Docker container; you should be fine leaving out uid and gid as it's sandboxed within it's own container with requests being handled by a separate Nginx or Apache container.

If you're concerned about this, you should be fine setting uid and gui as nobody, however it's not concrete and something I still need to test.

Read an interesting thread here about it

We'll cover running uWSGI behind another web server and from within a docker container shortly.

Let's explore some more configuration options:

strict = true
buffer-size = 65535
strict = true - This option tells uWSGI to only allow valid uWSGI options in the config file
buffer-size = 65535 - Increases the default buffer size (4096 bytes) for the headers of each request, minimizing "invalid request block size" errors
vacuum = true
die-on-term = true
vacuum = true - Removes the socket when the process stops
die-on-term = true - If running the uWSGI service with the init system (systemd, upstart), confirms both uWSGI and the init system have the same idea about what each process signal means
Basic logging
So far, uWSGI has been logging everything to the terminal.

There's lots of logging options available for uWSGI, but for now, we're just going to log everything to a file.

logto = uwsgi.log
This will log everything to a file named uwsgi.log (You need to create it first) and will look the same as what's been logged to the terminal so far.

I'm still trying to figure out my proferred logging mechanism for uWSGI logs and will update this guide at a later date. The problem with this approach is the log file will keep on growing indefinitely.

All together:

[uwsgi]

http = :8080

uid = nobody
gid = nobody

wsgi-file = run.py
callable = app

master = true
processes = 4
threads = 8

stats = stats.sock
memory-report = true

strict = true
buffer-size = 65535

vacuum = true
die-on-term = true

logto = uwsgi.log
uWSGI & Nginx on a virtual machine
A common pattern for deploying applications on virtual machines if to use Nginx as the web server, proxying requests to uWSGI which invokes the application itself.

Web client (Browser/application) <-> Web server (Nginx/Apache) <-> Socket <-> uwsgi <-> Python application (Flask/Django)
We'll make some changes to the ini file we created in the previous example to account for Nginx handling the incoming requests.

To keep clear separation from the previous example, we're going to rename http.ini to app.ini

The first thing we'll do is change http to socket as we don't require uWSGI to act as our client facing web server:

; http = 8080
socket = app.sock
Invoking the uwsgi command will create a socket file called app.sock in the same directory as the ini file.

A socket file is a special file used for inter-process communication, to which Nginx will commnunicate with whenever a request comes in from a client.

We'll add a new command to our ini file to change the permissions of the socket file:

chmod-socket = 664
We're using Nginx which defaults to the www-data user and group (On Ubuntu, may be different on other systems), so we'll also change the values for uid and gid to factor for this:

uid = www-data
gid = www-data
All together:

[uwsgi]
socket = app.sock
chmod-socket = 664

uid = www-data
gid = www-data

wsgi-file = run.py
callable = app

master = true
processes = 4
threads = 8

stats = stats.sock
memory-report = true

strict = true
buffer-size = 65535

vacuum = true
die-on-term = true

logto = uwsgi.log
Nginx server block
In order for Nginx to proxy requests to our app.sock file, we need to create a new server block. Here's a very basic example:

server {

    listen 80;

    location / {
        include uwsgi_params;
        uwsgi_pass unix:/path/to/app.sock;
    }
}
This server block will listen for requests on port 80 and pass them to our app.sock file. This file is typically found in a file such as /etc/nginx/sites-available/app and needs to be linked with the /etc/nginx/sites-enabled directory.

Assuming you've created a new file called app in /etc/nginx/sites-available/, you can do so with:

sudo ln -s /etc/nginx/sites-available/app /etc/nginx/sites-enabled
You can read the full tutorial on how to deploy a Flask application with uWSGI and Nginx on a VM here

uWSGI & Nginx in containers
Another popular pattern is to use Docker, with dedicated containers for Nginx and uWSGI (A Python image with uWSGI installed and containing your Flask/Django application)

Our app.ini file and Nginx config will look fairly similar to the previous example, with some minor changes.

We'll also setup some of the networking options in a docker-compose.yml file, exposing internal and external ports.

Let's start out by taking a look at the docker-compose.yml file:

version: "3.7"

services:

  nginx:
    build: ./nginx
    container_name: nginx
    restart: always
    ports:
      - "80:80"

  flask:
    build: ./flask
    container_name: flask
    restart: always
    expose:
      - 9090
The important parts to pay attention to here are ports and expose.

We're opening port 80 on the Nginx container and mapping it to the internal port 80, allowing Nginx to listen for incoming client requests.

In the Flask container, we're exposing port 9090 internally to any other containers on our network.

expose opens internal ports and are not accessible from the outside world, as opposed to ports which is used to map external ports to internal ports.

We're going to skip some of the other files, but you can find the source code for reference here on Github.

In our uWSGI ini file, we're going to change the socket value from app.sock to :9090.

; socket = app.sock
socket = :9090
We've also removed the uid and gid options.

[uwsgi]
wsgi-file = run.py
callable = app

socket = :9090
chmod-socket = 664

processes = 4
threads = 2
master = true

vacuum = true
die-on-term = true
Nginx Docker config
Just as on a virtual machine, we're going to use Nginx to proxy incoming requests to our app, however this time we're proxying to a port on the container hosting the application and uWSGI:

server {

    listen 80;

    location / {
        include uwsgi_params;
        uwsgi_pass flask:9090;
    }

}
As we've named the Flask container flask in our docker-compose.yml file, we can use it as the container hostname, followed by the port number.

Any incoming requests on port 80 will be proxied to the flask container on port 9090, where uWSGI will be listening and ready to invoke our application.

Wrapping up
This guide should give you a basic insight of how to use uWSGI in a few different scenarios and an introduction to some of the configuration options and performance monitoring tools.

With uWSGI being such a big project, I've skipped over many things, however you should definitely read more over at the official docs linked below:

The uWSGI project



