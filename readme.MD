# Installing WordPress on Dokku

To get WP up and running you have to prepare several things. This guide is based on a Dokku installation on a DigitalOcean Dropplet; for me the most convenient way to get a Dokku platform running.

#### Prepare your dokku environment
To make sure everything is on place, you should prepare your dokku environment.

1. Add your app
Create a app with the  dokku apps:create <appname> where appname is the sub domain of your dokku server.

2. Add a Database
I’ll use the official mariadb plugin for firing up a mysql database.

First add a new database by executing the following command  dokku mariadb:create .

Choose whatever you want for the databasename.

After firing the command, you should see a output like this:


    dokku@myserver:~# dokku mariadb:create my_wp_database
    -----> Starting container
       Waiting for container to be ready
    =====> MariaDB container created: my_wp_database
       DSN: mysql://mariadb:2995cee44058d534@dokku-mariadb-my-wp-database:3306/my_wp_database


Please note the line
  `DSN: mysql://mariadb:2995cee44058d534@dokku-mariadb-my-wp-database:3306/my_wp_database` we will need this information later to configure the WordPress database!
Now connect your database to your app. This step makes sure your app has access to the db.

dokku mariadb:link my_wp_database yourappname


##### Prepare the app configuration

Now we have to set some environment variables and docker options to make sure that WordPress is able to connect to the database, to crypt some cookies and to persist your data (uploads, themes, plugins). This is a very important step to make sure your data will be in place even after a restart of the container or the server.

##### Set the config

First start with the database configuration. You remember the DSN you got after you created the Database?

`DSN: mysql://mariadb:2995cee44058d534@dokku-mariadb-my-wp-database:3306/my_wp_database`

Lets split the string into some useable variables and set these for the app. Please execute the following commands (keep in mind that your appname and database credentials are different, so make sure you replace them):


    dokku config:set yourappname DB_HOST=dokku-mariadb-my-wp-database
    dokku config:set yourappname DB_NAME=my_wp_database
    dokku config:set yourappname DB_PASSWORD=2995cee44058d534
    dokku config:set yourappname DB_USER=mariadb

Next are the salt values for several security features of WordPress

    dokku config:set yourappname AUTH_KEY=whatyouwant
    dokku config:set yourappname AUTH_SALT=whatyouwant
    dokku config:set yourappname LOGGED_IN_KEY=whatyouwant
    dokku config:set yourappname LOGGED_IN_SALT=whatyouwant
    dokku config:set yourappname NONCE_KEY=whatyouwant
    dokku config:set yourappname NONCE_SALT=whatyouwant
    dokku config:set yourappname SECURE_AUTH_KEY=whatyouwant
    dokku config:set yourappname SECURE_AUTH_SALT=whatyouwant
    
Do a  dokku config. This gives you a list of all environment variables set for your app


    dokku@myserver:~# dokku config yourappname
    =====> yourappname config vars
    AUTH_KEY:         whatyouwant
    AUTH_SALT:        whatyouwant
    DATABASE_URL:     mysql://mariadb:2995cee44058d534@dokku-mariadb-my-wp-database:3306/my_wp_database
    DB_HOST:          dokku-mariadb-my-wp-database
    DB_NAME:          my_wp_database
    DB_PASSWORD:      2995cee44058d534
    DB_USER:          mariadb
    LOGGED_IN_KEY:    whatyouwant
    LOGGED_IN_SALT:   whatyouwant
    NONCE_KEY:        whatyouwant
    NONCE_SALT:       whatyouwant
    SECURE_AUTH_KEY:  whatyouwant
    SECURE_AUTH_SALT: whatyouwant
    
Everything in place? Then move on 

#### Configure the docker-options

This Step is very important to preserve uploads and downloaded themes and plugins. If you miss these steps, your files will we gone after a restart of the container.
 Just create a directory on your dokku host. For my example, i created the directory /opt/wordpress/.

##### Then link the directory to your container:

    dokku docker-options:add yourappname deploy,run "-v /opt/wordpress:/app/wp-content"

Thats all, the environment is prepared now. Let’s configure and deploy WordPress 🙂

#### Prepare WordPress
Just clone this repository on your local machine.

    git clone https://github.com/imana97/wordpress-dokku.git

 
#### Deploy your app

Now we have to add your dokku server as a git remote (remember to change the dokku server and hostname)

    git remote add dokku dokku@myserver.com:myhostname

And now, DEPLOY

    git push dokku master
