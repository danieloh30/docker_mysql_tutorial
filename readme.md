Docker with Docker Toolbox for Windows Java EE + MySQL
=======================================================

This tutorial walks you through using a Java EE application server and MySQL, both running in Docker Linux containers on a Windows host via Docker Toolbox.

* * *
##### Prerequisites & Assumptions:
* You completed the first [Docker Tutorial](https://github.com/burrsutter/docker_tutorial) that walks through the basics of standing up a Java EE application server in a Docker container
* You have [MySQL Workbench](http://dev.mysql.com/downloads/workbench/) installed on your Windows host
* You have [Maven](http://maven.apache.org/) and a [JDK](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) installed on your Windows host machine
* Note: We give you an out if Java & Maven are not working well for you
* * *


1. Docker Quickstart terminal

    ![Alt text](/screenshots/docker_quickstart_terminal.png?raw=true "Start Menu")

    ![Alt text](/screenshots/start_sh_running.png?raw=true "Boot2Docker Command Prompt")

2. `docker network create mysqlapp_net`

    > This command will create a network that will be used by mysql and wildfly containers

    ![Alt text](/screenshots/docker_network_create.png?raw=true "docker network create mysqlapp_net")


3. Now we run MySQL using the recently created network, inside of a Docker container

    ````
    docker run --name mysqldb --net mysqlapp_net -p 3306:3306 -e MYSQL_USER=mysql -e MYSQL_PASSWORD=mysql -e MYSQL_DATABASE=sample -e  MYSQL_ROOT_PASSWORD=supersecret -d mysql
    ````

    This will take some time if this is the first run of the "mysql" image

    ![Alt text](/screenshots/docker_run_mysql.png?raw=true "docker run mysql")

    > Note: I am using -p 3306:3306 to connect from a Windows (host) MySQL Workbench and from a Windows/host based Wildfly instance. 3306 is the normal port for MySQL.  You will see this port listed in the `docker ps` results as well.

    * * *
    ##### Tip: Copy & Paste in the default Windows Command Prompt is possible.
    Click on the top-left icon for the pull-down menu.  Select Mark, highlight the text you wish to copy, then hit Enter. Then for Paste, simply use the menu option.
    ![Alt text](/screenshots/mark_paste.png?raw=true "DOS/Command Prompt Copy & Paste")

    * * *

4. Back in Windows run MySQL Workbench.  You will need the IP address you saw when running `Docker Quickstart terminal`  or `docker-machine ip default`.

    > Select New Connection

    ![Alt text](/screenshots/mysql_workbench_new_connection.png?raw=true "new connection")

    > Give the connection a name like "docker_mysql_104" as the IP address will change from time to time. Enter the correct IP address.  You will notice that some of the screenshots use different IPs.

    ![Alt text](/screenshots/connect_to_database.png?raw=true "New Connection Dialog")

    > The root password was defined via the `docker run` command, look for MYSQL_ROOT_PASSWORD

    ![Alt text](/screenshots/mysql_root_password.png?raw=true "Root Password Prompt")

    > Double-click on the connection to open up the SQL Editor

    ![Alt text](/screenshots/mysql_sql_editor.png?raw=true "SQL Editor")


    > By default you get a "sample" database with no tables.  A table will be created when we launch the Java EE application.

    > Now that your MySQL instance is running happily inside of a Docker container (inside the docker host called default)
let's configure the Java EE app to use your MySQL

5. In the first tutorial, we gave you the .war pre-built, in this case, we want you to download the project sources to your local machine and perform you own Maven build to generate the .war.

    > Download and unzip <https://github.com/burrsutter/docker_mysql_tutorial/archive/master.zip>

    ![Alt text](/screenshots/download_unzip.png?raw=true "Download and Unzip")


6.  `mkdir /c/Users/<your_username>/docker_projects/mysqlapp`

    > Remember docker hosts have VirtualBox Guest Additions preconfigured to map the Linux `/c/Users/` to `c:\Users`
    > You can also just use the Windows Explorer to create this directory

7. Using File Explorer, copy the directory `javaee6angularjsmysql` to mysqlapp

8. From a Windows (DOS) Command Prompt,
    ````
    cd \Users\<your_username>\docker_projects\mysqlapp\javaee6angularjsmysql
    mvn clean compile package
    ````


    > This does require that you have a JDK and Maven installed plus your PATH set properly.  Setting up your PATH can be as simple as executing the following commands:

    ````
    set JAVA_HOME=C:\tools\java\jdk1.7.0_04_32bit
    set M2_HOME=C:\tools\apache-maven-3.2.3
    set PATH=%JAVA_HOME%\bin;%M2_HOME%\bin;C:\tools;%PATH%
    ````

    > Look for BUILD SUCCESS

    ![Alt text](/screenshots/build_success.png?raw=true "mvn clean compile package")

    > Do not have Maven nor the Java Development Kit running well on this workstation? No problem, just download the pre-built .war from <https://github.com/burrsutter/docker_tutorial/blob/master/javaee6angularjsmysql.war?raw=true>

9. Open the javaee6angularjsmysql project in your favorite editor (I have been trying out Atom.io) and review the persistence.xml file (located in src/main/resources/META-INF folder).  Take note of the following line:

    ````
    <jta-data-source>java:jboss/datasources/MySQLSampleDS</jta-data-source>
    ````

    ![Alt text](/screenshots/persistence_xml.png?raw=true "persistence.xml")

    > This Java EE6 application needs a datasource which must be configured inside of the application server.  The datasource needs a JDBC Driver to have been pre-loaded as well.

10.  Copy the file MySQL JBDC driver - mysql-connector-java-5.1.31-bin.jar - to your mysqlapp directory.  When the docker image is created, this file will be dropped into Wildfly's standalone\deployments directory.  Wildfly hot deploys JDBC drivers it finds in its deployments directory.


11.  Copy the file mysql-sample-ds.xml to your mysqlapp directory and open it in your editor.

    ![Alt text](/screenshots/mysql_sample_ds_xml.png?raw=true "mysql-sample-ds.xml")

    > Now, this is where a lot of the magic happens.  This file when found in Wildfly's standalone\deployments directory will configure a Datasource inside of Wildfly.  You should notice that the JNDI name of "MySQLSampleDS" matches what you have in the persistence.xml.  The hosname, user and password for MySQL were setup back with the `docker run` command in step 3.

    ````
    <connection-url>jdbc:mysql://mysqldb:3306/sample</connection-url>
    <driver>mysql-connector-java-5.1.31-bin.jar_com.mysql.jdbc.Driver_5_1</driver>
    <security>
      <user-name>mysql</user-name>
      <password>mysql</password>
    </security>
    ````

12. Copy the standalone.xml and Dockerfile to your mysqlapp directory and open it up in your editor.

    ````
    FROM centos/wildfly

    COPY standalone.xml /opt/wildfly/standalone/configuration/
    COPY mysql-connector-java-5.1.31-bin.jar /opt/wildfly/standalone/deployments/
    COPY mysql-sample-ds.xml /opt/wildfly/standalone/deployments/

    COPY javaee6angularjsmysql/target/javaee6angularjsmysql.war /opt/wildfly/standalone/deployments/

    ````
    ![Alt text](/screenshots/mysqlapp_with_Dockerfile.png?raw=true "mysqlapp directory with Dockerfile")

    > mysql-connector-java-5.1.31-bin.jar is in the same directory as the Dockerfile

    > mysql-sample-ds.xml and standalone.xml are also in the same directory

    > javaee6angularjsmysql.war was created via `mvn clean compile package`

    ![Alt text](/screenshots/target_directory.png?raw=true "target directory")


    > Note: This is NOT how you would normally configure a JDBC driver and Datasource for a production ready Docker image.  You are likely separate these into three different layers/images - allowing you to update the JDBC driver in a single layer/location/image.


13. Build the new Docker image in the `Docker Quickstart terminal`

    ````
    cd /c/Users/<your_username>/docker_projects/mysqlapp

    docker build --tag=mysqlapp .
    ````

    ![Alt text](/screenshots/after_docker_build.png?raw=true "docker build results")


14. `docker run -it --net mysqlapp_net -p 8080:8080  mysqlapp`

    > The MySQL container was started with "--name mysqldb" back in Step 3 and the "mysqldb" is referenced in the -ds.xml as a hostname. You can verify the "mysqldb" name of the MySQL container via the `docker ps` command.
    >
    > You can find out more information about Docker networks in the official documentation at
    <https://docs.docker.com/engine/userguide/networking/dockernetworks/>



15. Use your browser to interact with the application, register a new Member.  You will need the IP address you saw when running `Docker Quickstart terminal`  or `docker-machine ip default`.

    ![Alt text](/screenshots/browser.png?raw=true "Application in Browser")

    > Then check out your MySQL database in the SQL Editor. Note that the table name is case sensitve.

    ![Alt text](/screenshots/sql_editor.png?raw=true "SQL Editor")


#### Extra Credit

To enable the Wildfly Admin Console, add the following to your Dockerfile

````
RUN /opt/wildfly/bin/add-user.sh admin Admin!1234 --silent
````

and build the image again:

````
docker build --tag=mysqlapp .
````

This is an example where the Docker build will execute a screen to provide custom configuration of the resulting Docker image.  In this specific case "admin" is the user id and "Admin!1234" is the password.
You could have used this same technique to execute commands that would have configured the JDBC driver and Datasource inside of the container.  We choose the -ds.xml and hot deployment of the JDBC driver .jar techniques above due to 'ease of learning'.

The JBoss Wildfly Admin Console uses a different port, so also change your "docker run" statement as follows:

````
docker run -it -p 8080:8080 -p 9990:9990 --net mysqlapp_net mysqlapp
````

![Alt text](/screenshots/wildfly_admin_console.png?raw=true "JBoss Wildfly Admin Console")

The End
