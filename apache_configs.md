# Apache Configurations
## Installing WSGI on Apache:
Apache requires mod_wsgi to execute wsgi apps like Flask

1. Check if mod_wsgi is installed

    The command `httpd -M | grep wsgi` will return `wsgi_module (shared)` if mod_wsgi is installed

2. Install mod_wsgi
    \# search for available mod_wsgi versions

    `yum search wsgi`

    \# install mod_wsgi if necessary. make sure the version of python matches the version used in development

    `sudo yum install mod24_wsgi-python36.x86_64`

3. Create app folder at `/var/www/your-app`

4. Change ownership of your-app to the current user
    \# the default owner of any folders or files created in `/var/www/` is root

    `chown -R $USER:$USER /var/www/your-app`

5. Create your virtualenv in your-app and install necessary dependencies. DO NOT install as sudo. sudo will install globally instead of to your virtualenv.

6. Create your flask app and a wsgi file. The name of the wsgi file can be anything.
    \# the contents of the wsgi file should be as follows:

    ```
    import os
    import sys
    import site

    # Add virtualenv site packages
    # Make sure to change the the url to match <your venv name>/local/lib64/<your venv python version>/site-packages
    site.addsitedir(os.path.join(os.path.dirname(__file__), 'env/local/lib64/python2.7/site-packages'))

    # Path of execution
    sys.path.append('/var/www/your-app')

    # import app as application
    from <your-flask-app-package-name> import app as application
    ```

7. Create virtual host config file and save at /etc/httpd/conf.d/
    \# create the file
        `sudo vim /etc/httpd/conf.d/vhost.conf`

    The content of `vhost.conf` should be as follows:
    ```
    <VirtualHost *:80>
        ServerName your.public.aws.dns.com

        # Create the WSGIDaemonProcess
        # The name of the process can be anything you like. In this case it is webtool.
        # Unless you changed it, the default user and group will be apache
        # If using virtual environment you need to specify that directory in python-home

        WSGIDaemonProcess webtool user=apache group=apache threads=5 python-home=/var/www/your-app/your-venv

        # Define any environment variables required by the app here
        # The app will not be able to access any environment variables defined in the OS
        # Do not use quotes for the literal value

        SetEnv MY_VARIABLE somevalue

        # Define the WSGIScriptAlias
        # This defines the location where your-app can be accessed.
        # By default it is availble at your.public.aws.dns.com
        # If you want to make it available in a sub folder like your.public.aws.dns.com/mycoolapp
        # use WSGIScriptAlias /mycoolapp /var/www/your-app/deploy.wsgi

        WSGIScriptAlias / /var/www/your-app/deploy.wsgi

        <directory /var/www/your-app>
                WSGIProcessGroup webtool
                WSGIApplicationGroup %{GLOBAL}
                WSGIScriptReloading On
                Order deny,allow
                Allow from all
        </directory>
    </VirtualHost>
    ```

8. Update relative paths in any files
    If the original app ran at localhost:5000 and now runs in a sub directory like `your.public.aws.dns.com/mycoolapp`, any calls in your JS files to make request using relative paths like `$getJSON('/resource1/')` will need to be updated to include the new subdirectory `$getJSON('/mycoolapp/resource1')`.

## Installing Mod_JK
1. Install mod_jk tomcat connector for Apache (https://tomcat.apache.org/connectors-doc/webserver_howto/apache.html)
    ```
    cd /home/ec2-user
    mkdir mod_jk
    cd mod_jk

    # downlad
    wget http://mirror.nexcess.net/apache/tomcat/tomcat-connectors/jk/tomcat-connectors-1.2.42-src.tar.gz

    # extact
    tar xzf tomcat-connectors-1.2.42-src.tar.gz

    # build
    cd tomcat-connectors-1.2.40-src/native
    ./configure --with-apxs=/usr/sbin/apxs
    make

    # install
    sudo make install
    ```

2. In order to use the mod_jk module the following needs to be appended to the file /etc/httpd/conf/httpd.conf
    ```
    # Load mod_jk module
    # Update this path to match your modules location
    LoadModule    jk_module  /usr/lib64/httpd/modules/mod_jk.so

    # Where to find workers.properties
    # Update this path to match your conf directory location (put workers.properties next to httpd.conf)
    JkWorkersFile /etc/httpd/conf/workers.properties

    # Where to put jk shared memory
    # Update this path to match your local state directory or logs directory
    JkShmFile     /var/log/httpd/mod_jk.shm

    # Where to put jk logs
    # Update this path to match your logs directory location (put mod_jk.log next to access_log)
    JkLogFile     /var/log/httpd/mod_jk.log

    # Set the jk log level [debug/error/info]
    JkLogLevel debug

    # Select the timestamp log format
    JkLogStampFormat "[%a %b %d %H:%M:%S %Y] "
    ```

3. Define workers.properties file
    ```
    # create new file
    sudo vim /etc/httpd/conf/workers.properties

    # Add geoserver to the list of workers
    worker.list=geoserver_worker

    # Define geoserver_worker
    worker.geoserver_worker.port=8009
    worker.geoserver_worker.host=localhost

    # uses the ajpv13 protocol to forward requests to a Tomcat process.
    worker.geoserver_worker.type=ajp13
    ```

4. Modify virtual host directives by adding the following to your virtual host directives
    ```
    JKMountCopy on
    # associate a url with a worker, for example
    # JKMount /sub sub_woker will forward requests to http://your-domain.com/geoserver to the sub_worker defined in worker.properties file.
    JKMount /<sub-domain-name> <worker_name>

    # associate all sub-url with a worker
    JKMount /<sub-domain-name>/* <worker_name>
    ```

## Enable TSL/SSL for HTTPS

1. Install required dependencies
* Make sure httpd is running

    `chkconfig --list httpd`

* Update the instance

    `sudo yum update -y`

* Install mod24_ssl

    `sudo yum install -y mod24_ssl`

* Restart httpd

    `sudo service httpd restart`

2. Update the virtual host to redirect all traffic to HTTPS (Optional)
    ```
    <VirtualHost *:80>
        ServerName <your-aws-public-dns>
        # redirect all traffic on port 80 (http) to port 443 (https)
        Redirect permanent / https://<your-aws-public-dns>
    </VirtualHost>

    <VirtualHost _default_:443>
        ServerName <your-aws-public-dns>
        Other directives...
    </VirtualHost>
    ```

# Geoserver Configurations
## Installing Geoserver on Tomcat
To install geoserver with Tomcat

1. Install Tomcat

    `sudo yum install tomcat7 tomcat7-webapps tomcat7-admin-webapps`

    Default java in tomcat7 is 1.7 Geoserver requires 1.8
    Need to update/change before geoserver will work. To update:

    ```
    sudo yum install java-1.8.0
    sudo yum remove java-1.7.0-openjdk (whatever other version you have. optional)
    ```

2. Install Geoserver
    ```
    cd /home/ec2-user/
    # download
    wget https://sourceforge.net/projects/geoserver/files/GeoServer/2.12.1/geoserver-2.12.1-war.zip
    # unzip
    unzip geoserver-2.12.1-war.zip
    # change owner and group to tomcat
    sudo chown tomcat:tomcat geoserver.war
    # move the war file into tomcat webapps folder
    sudo mv geoserver.war /var/lib/tomcat7/webapps/
    ```

### Proxy from Apache to Tomcat
1. Install mod_jk, if it hasn't been installed.

    See [Installing Mod_JK](#installing-mod_jk) section for instruction

2. Load mod_jk into apache httpd.conf file if it hasn't been loaded.

    See [Installing Mod_JK](#installing-mod_jk) section for instruction

3. Modify virtual host directives by adding the following, check that the worker name match what is defined in `workers.properties` file.

    ```
    # if using mod_jk to connect to services from tomcat e.g. geoserver
    # the JKMount commands must be added here in the virtual host defintion instead of httpd.conf
    # delete the 3 lines below if mod_jk is not being used
    JKMountCopy on
    JKMount /geoserver geoserver_worker
    JKMount /geoserver/* geoserver_worker
    ```

4. When running Geoserver on Tomcat and proxying thru Apache, you need to update your global proxy settings.

    * Log in as admin ==> Settings ==> Global
    * Set Proxy Base URL to your service i.e. `www.myserver.com/geoserver`
