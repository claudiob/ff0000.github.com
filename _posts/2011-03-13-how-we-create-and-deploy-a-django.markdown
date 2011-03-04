---
layout: post
title: "How we create and deploy a Django project to the Rackspace cloud"
author: claudiob
---

Frequently we need to create a Django project from scratch and push it on a production machine. This post explains how we have reduced the number of manual steps to achieve this goal.

## Creating the blank project

We use our own pypeton application to create a Django project that matches our coding standard and set of requirements:

    ./pypeton fooapp


This will create a *fooapp* folder with the default branches/tags/trunk folder for subversion. The trunk folder will contain a set of configuration folders and files and a Django project in the *project* folder.

## Installing requirements

The next step is to install the requirements for the project, with these commands:

    cd fooapp/trunk
    virtualenv .
    source bin/activate
    bin/pip install -r deploy/requirements.txt

## Running the project in the browser

The project already comes with a dummy application names *things*, a model with just a *name* and a *slug*. This application can be seen in a web browser by running:

    cd project
    python manage.py syncdb
    python manage.py runserver 0.0.0.0:8000 --adminmedia=static/admin

and then opening [http://localhost:8000](http://localhost:8000) in a browser window.

## Loading fixtures

The home page will show the view corresponding to the *list of things*, which is empty so far as we haven't entered any data. We can easily fix this by loading the fixtures that come with the application:

    python manage.py loaddata test

and restarting the server, to see two sample things appearing in the home-page.

## Customizing the sample model

We can customize the project and change *things* for something else. For instance, if our project needs a *movie* model rather than a *thing* one we simply replace any occurrence of thing, things and Thing in the model with movie, movies and Movie. We also have to rename the folders and files appropriately:

<!-- I need help here with the bash command -->

Then we can edit the newly named model and add fields if required, for instance having the *run_time* field added to the Movie class in apps/movies/models.py:

{% highlight python %}
class Movie(models.Model):
    name             = models.CharField(unique=True, max_length=255)
    slug             = models.SlugField(unique=True, max_length=255)
    run_time         = models.IntegerField(blank=True, null=True)
{% endhighlight %}

Similarly, we have to update the test fixtures to reflect this change in apps/movies/fixtures/test.json:

{% highlight javascript %}
[
    {
        "pk": 1, 
        "model": "movies.movie", 
        "fields": {
            "name": "Strange Days", 
            "slug": "strange-days", 
            "run_time": "145" 
        }
    },
    {
        "pk": 2, 
        "model": "movies.movie", 
        "fields": {
            "name": "My Life as a Dog", 
            "slug": "my-life-as-a-dog", 
            "run_time": "101" 
        }
    }
]
{% endhighlight %}

And then reload the database and relaunch the server:

    rm *.db
    python manage.py syncdb
    python manage.py loaddata test
    python manage.py runserver 0.0.0.0:8000 --adminmedia=static/admin

## Running the integration suite

The project created by pypeton also come with a basic integration test suite. The suite can be run by creating an appropriate test database and then running the lettuce command against it for the application:

    python manage.py syncdb --settings=settings_test
    python manage.py harvest --settings=settings_test -d

This will pop up an instance of Firefox and execute the basic scenarios specified in apps/movies/features, simply that a movie can be added through the admin interface and that its name will show app in the show page.

## Committing to subversion

The project is now ready to be committed to subversion. Create a remote repository and push the code:

    ssh [user]@[svn_server]
    sudo /opt/red/bin/make_svn fooapp
    exit
    cd ../../ # go back to the fooapp root
    svn co svn+ssh://[user]@[svn_server]/svn/fooapp .
    svn add *

<!-- Here I need help in adding svn:ignore for the virtualenv folders!
*.pyc
.Python
bin
include
lib
build
src
 -->

    svn ci -m"Basic fooapp project with a 'movie' model."

## Creating the server instance on Rackspace

To move the code into production, we create a new instance on Rackspace by using the [rscurl](https://github.com/jsquared/rscurl) command line tool:

    cd .. # go back to the pypeton folder
    ./rscurl -u ff0000 -a [app_key] -c create-server -i 49 -f 1 -n fooapp-dev


where *ff0000* is our Rackspace cloud username and the *app_key* can be obtained from the settings panel on Rackspace. The previous command line generates a "Ubuntu 10.04 LTS (lucid)" machine with 256MB.

## Adding Rackspace data to the deployment code

Creating a server instance on Rackspace returns an admin password, a public IP and a private IP, which we replace in the default code created by pypeton. Specifically, we:

- edit trunk/deploy/nginx.conf and write the private IP where "server 127.0.0.1:8001" appears and the public IP where "server_name  127.0.0.1" appears
<!-- NOTE: this should be extracted from the rscurl response -->
- edit trunk/deploy/uwsgi.ini and write the private IP where "socket = 127.0.0.1:8001" appears
<!-- NOTE: this should be extracted from the rscurl response -->

## Setting up the server instance on Rackspace

At this point we follow the instructions for [django-fab-deploy](https://github.com/ff0000/red-fab-deploy):

    ssh-keygen -f fooapp_rsa -N ""
    mv fooapp_rsa* fooapp/trunk/deploy/
    mv fooapp/trunk/src/red-fab-deploy/fabfile_example.py fooapp/trunk/fabfile.py

We replace into fooapp/trunk/fabfile.py the values of INSTANCE\_NAME to fooapp, the value of REPO to [svn_server]/fooapp and the value of SERVERS['DEV'] to ubuntu@[public\_ip]. Then we deploy the current trunk by running:
<!-- NOTE: this should be extracted from the rscurl response -->

    cd fooapp/trunk
    fab dev rackspace_as_ec2:"deploy/fooapp_rsa.pub"

We enter the Rackspace admin password when required <!-- NOTE: this should be extracted from the rscurl response --> and return every question but "Enter the group name you want to add the user to" where we specify *www-data*

## Deploying from subversion to Rackspace

We deploy the trunk of the subversion repository to Rackspace running:

    fab dev full_deploy:"trunk" -i deploy/fooapp_rsa 

and answering "yes" to "Is the OS detected correctly (lucid)?". This step takes a while since all the Python requirements are installed on the server. When the machine is about to retrieve the code from subversion, enter your SVN credentials (just press ENTER when *Password for 'ubuntu'* comes up) and specify not to store password unencrypted.

## Launching the application on Rackspace

We start the nginx server and uWSGI on Rackspace running:

    fab dev web_server_setup web_server_start -i deploy/[your private SSH key here]

Then we execute the commands to set up the project database on the production machine:

    fab dev syncdb -i deploy/[your private SSH key here]
    fab dev manage:"loaddata test" -i deploy/[your private SSH key here]

Finally open up the public IP in the browser and see the application running in production!

## Updating the code and restarting the server

When we have to update the code on the production machine, we <!-- NOTE: This step is missing, we should have a way for fab to automatically create a tag when a project is deployed, then deploy that tag and change the /active symlink on the server. Once we have the tag, the commands run by fab would be -->:

    fab dev deploy_project:"tagname" -i deploy/[your private SSH key here] 
    fab dev make_active:"tagname" -i deploy/[your private SSH key here] 

Finally, the web server can be restarted by running:

    fab dev uwsgi_restart -i deploy/[your private SSH key here] 
    fab dev web_server_restart -i deploy/[your private SSH key here] 
