# Installing WordPress on Dokku

To get WP up and running you have to prepare several things. This guide is based on a Dokku installation on a DigitalOcean Dropplet; for me the most convenient way to get a Dokku platform running.

Prepare your dokku environment
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


#####Prepare the app configuration

Now we have to set some environment variables and docker options to make sure that WordPress is able to connect to the database, to crypt some cookies and to persist your data (uploads, themes, plugins). This is a very important step to make sure your data will be in place even after a restart of the container or the server.

#####Set the config

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
    
Do a  dokku config . This gives you a list of all environment variables set for your app


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

####Configure the docker-options

This Step is very important to preserve uploads and downloaded themes and plugins. If you miss these steps, your files will we gone after a restart of the container.
 Just create a directory on your dokku host. For my example, i created the directory /opt/wordpress/.

#####Then link the directory to your container:

`dokku docker-options:add yourappname deploy,run "-v /opt/wordpress:/app/wp-content"`

Thats all, the environment is prepared now. Let’s configure and deploy WordPress 🙂

####Prepare WordPress
Just clone the current WordPress repository from Github and add a deployment branch:


    git clone https://github.com/WordPress/WordPress wordpress
    cd wordpress
    git checkout -b deployment

 

First add a file called composer.json to your repository. This will make sure that the correct version of php will be used and also install some basic libraries like gd and curl.


    {
      "require": {
        "php": "~5.6.0",
        "ext-mbstring" : "*",
        "ext-gd": "*",
        "ext-curl": "0.0.0.*",
        "ext-exif": "0.0.0.*"
      },
      "scripts": {
        "post-install-cmd": [
          "chmod -R 777 wp-content"
        ]
      }
    }

 

Next add a file called nginx_app.conf. This configures the nginx service for your subdomain/app.



    # WordPress permalinks
    location / {
      index index.php index.html;
      try_files $uri $uri/ /index.php?$args;
    }
     
    # Add trailing slash to */wp-admin requests.
    rewrite /wp-admin$ $scheme://$host$uri/ permanent;
     
    # Deny access to any files with a .php extension in the uploads directory
    # Works in sub-directory installs and also in multisite network
    location ~* /(?:uploads|files)/.*.php$ {
      deny all;
    }
     
    #upload
    client_max_body_size 100M;
     
    #jetpack connection
    fastcgi_buffers 8 32k;
    fastcgi_buffer_size 64k;
    proxy_buffer_size 128k;
    proxy_buffers 4 256k;
    proxy_busy_buffers_size 256k;
     
    # enable gzip compression
    gzip on;
    # Minimum file size in bytes (really small files aren’t worth compressing)
    gzip_min_length 1000;
    # Compression level, 1-9
    gzip_comp_level 2;
    gzip_buffers 4 32k;
    gzip_types text/plain application/javascript text/xml text/css image/svg+xml;
    # Insert `Vary: Accept-Encoding` header, as specified in HTTP1.1 protocol
    gzip_vary on;
    # end gzip configuration
     
    # Set time to expire for headers on assets
    location ~* .(js|css|png|jpg|jpeg|gif|ico|svg)$ {
      expires 1y;
    }
     
    # Sitemap url, for WordPress SEO plugin
    #rewrite ^/sitemap_index.xml$ /index.php?sitemap=1 last;
    #rewrite ^/([^/]+?)-sitemap([0-9]+)?.xml$ /index.php?sitemap=$1&sitemap_n=$2 last;
     

I also added a file called custom_php.ini where you can set several php options for your deployment.

    upload_max_filesize = 50M
    post_max_size = 50M

Now add a file named `Procfile`. This will ensure dokku is loading your configuration.


`web: vendor/bin/heroku-php-nginx -C nginx_app.conf -i custom_php.ini --verbose`

Finally, copy the file wp-config.php.sample to wp-config.php and modify some variables:



    $_SERVER['SERVER_PORT'] = "80";
     
    // ** MySQL settings - You can get this info from your web host ** //
    /** The name of the database for WordPress */
    define('DB_NAME', getenv('DB_NAME'));
     
    /** MySQL database username */
    define('DB_USER', getenv('DB_USER'));
     
    /** MySQL database password */
    define('DB_PASSWORD', getenv('DB_PASSWORD'));
     
    /** MySQL hostname */
    define('DB_HOST', getenv('DB_HOST'));
     
    /** Database Charset to use in creating database tables. */
    define('DB_CHARSET', 'utf8');
     
    /** The Database Collate type. Don't change this if in doubt. */
    define('DB_COLLATE', '');
     
    /**#@+
     * Authentication Unique Keys and Salts.
     *
     * Change these to different unique phrases!
     * You can generate these using the {@link https://api.wordpress.org/secret-key/1.1/salt/ WordPress.org secret-key service}
     * You can change these at any point in time to invalidate all existing cookies. This will force all users to have to log in again.
     *
     * @since 2.6.0
     */
    define('AUTH_KEY',         getenv('AUTH_KEY'));
    define('SECURE_AUTH_KEY',  getenv('SECURE_AUTH_KEY'));
    define('LOGGED_IN_KEY',    getenv('LOGGED_IN_KEY'));
    define('NONCE_KEY',        getenv('NONCE_KEY'));
    define('AUTH_SALT',        getenv('AUTH_SALT'));
    define('SECURE_AUTH_SALT', getenv('SECURE_AUTH_SALT'));
    define('LOGGED_IN_SALT',   getenv('LOGGED_IN_SALT'));
    define('NONCE_SALT',       getenv('NONCE_SALT'));
    
If everything is done, commit the files and changes to our branch


    git add -A
    git commit -m "Dokku stuff"

 
####Deploy your app

Now we have to add your dokku server as a git remote (remember to change the dokku server and hostname)

git remote add dokku dokku@myserver.com:myhostname

And now, DEPLOY

    git push dokku deployment:master
