---
layout: post
title: "How we create and deploy a Django project to the Rackspace cloud"
author: claudiob
---

Frequently we need to create a Django project from scratch and push it on a production machine. This post explains how we have reduced the number of manual steps to achieve this goal.

## Creating the blank project

We use our own pypeton application to create a Django project that matches our coding standard and set of requirements:

    ./pypeton -v -s fooapp


This will create a *fooapp* folder with the default branches/tags/trunk folder for subversion. The trunk folder will contain a set of configuration folders and files and a Django project in the *project* folder. Do not rename the *project* folder to the name of your project, if you do the fabric deploy scripts will not work.

## Installing requirements

The next step is to install the requirements for the project, with these commands:

    cd fooapp/trunk
    source activate
    pip install -r deploy/requirements.txt

## Running the project in the browser

The project comes with an empty home-page which can be seen in a web browser by running:

    cd project
    python manage.py syncdb
    ./ser

and then opening [http://localhost:8000/](http://localhost:8000/) in a browser window. `./ser` is a shortcut provided for the command "python manage.py runserver 0.0.0.0:8000" used frequently to start the local server.

## Loading fixtures

The project also comes with an application named *initial_data*, a model that creates an *admin* user with password *admin* when in development mode and site called http://example.com:8000. The Django admin can be accessed by typing:

    python manage.py loaddata development

and restarting the server and logging into [http://example.com:8000/admin](http://example.com:8000/admin) after having add example.com to the local hosts file.

Note that the admin/admin user is *not* included in the initial_data.json file, to prevent creating this user by mistake on production machines.

## Creating a new model and application

A new model (e.g., called *Picture*) can be created with the command:

    ./sta Picture

which extends the default "python manage.py startapp" command by delivering a coherent file structure and providing two basic views for index and show. To include this model in the application:

* add `'pictures',` to `INSTALLED_APPS` in settings.py, and
* add `(r'^pictures/', include('pictures.urls'))` to `urlpatterns` in urls.py.
* run `python manage.py syncdb` to add the model to the database

At this point, all these new views become available:

* http://example.com:8000/admin/pictures/picture/add/ (to add a picture)
* http://example.com:8000/admin/pictures/picture/ (to admin the pictures)
* http://example.com:8000/pictures/ (to show the list of pictures)
* http://example.com:8000/pictures/1/ (to show one picture after its creation)

## Customizing the model

The newly created model can be edited at will. For instance, the Picture model can be inherited by [Photologue](http://code.google.com/p/django-photologue/)'s ImageModel in order to have many image-related functions available. For this purpose:

* add the following lines to deploy/requirements.txt:

        pil                   # to use models.ImageField
        # django-photologue   # to deal with image size, thumbnails
        # The default photologue does not deal with thumbnail generation on CDN
        -e git+git://github.com/ff0000/django-photologue@cumulus#egg=django-photologue

* run the command `./req` to install the new requirements

* add `'photologue',` to `INSTALLED_APPS` in settings.py

* edit apps/pictures/models.py to begin as:

    from django.db import models
    from photologue.models import ImageModel
    class Picture(ImageModel):

* add `<img src="{{picture.image.url}}" />` to the content in templates/pictures/show.html

## Resetting the database
 
The next step is to adjust the database in order the include the Photologue fields into the Picture table. Django 1.2 does not provide this command, but this can be obtained by simply running:

    ./syn development

where `./syn` is a shortcut to:

* reset the database structure 
* synchronize it again with the latest models
* optionally reload the fixtures for the specified environment (in this case, environment)

At this point, the following views will allow to:

* http://example.com:8000/admin/pictures/picture/add/ (add a picture with an attached image)
* http://example.com:8000/pictures/1/ (see the attached image of the inserted picture)

## Creating fixtures

Whenever the `./syn` command is run, all the data in the database is deleted. The method not to lose data during a synchronization is using fixtures.

For instance, after adding a Picture through the admin interface, this can be stored in a fixture running:

    ./dum pictures development

where `./dum` is a shortcut for `python manage.py dumpdata` to output the data with the right indentation in the specified environment.

At this point, running `.syn development` again will reset the database and reload the images from the fixtures, without losing any data in the operation.

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
