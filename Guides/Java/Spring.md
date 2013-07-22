#Deploying a Spring Application


In this tutorial we're going to show you how to create an example Spring/MVC/Hibernate application using Spring Roo. We will integrate it with the [MySQLs Add-on] and deploy it with an [embedded Jetty server] on [cloudControl]. 

You can find the [source code on Github]. Check out the [buildpack-java] for supported features.


##The Spring Application Explained

### Create the App

Install [Spring Roo] and use the `bin/roo.sh` script to create the "Petclinic" example application:

~~~bash
$ export PATH=SPRING_ROO_PATH/bin:$PATH
$ mkdir PROJECTDIR; cd PROJECTDIR;
$ roo.sh script --file clinic.roo
~~~

Generate data source configuration for [Hibernate] / MySQL

~~~bash
$ roo.sh persistence setup --provider HIBERNATE --database MYSQL
~~~

###Prepare to run on Jetty

The [Jetty Runner] provides a fast and easy way to run your app without maintaining a full JEE server. Simply add it to the build plugins section in your `pom.xml`:

~~~xml
...
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-dependency-plugin</artifactId>
            <version>2.3</version>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>copy</goal>
                    </goals>
                    <configuration>
                        <artifactItems>
                            <artifactItem>
                                <groupId>org.mortbay.jetty</groupId>
                                <artifactId>jetty-runner</artifactId>
                                <version>7.4.5.v20110725</version>
                                <destFileName>jetty-runner.jar</destFileName>
                            </artifactItem>
                        </artifactItems>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
~~~

###Adjust Data Source Configuration to MySQLs Add-on

Go to the application context configuration file `src/main/resources/META-INF/spring/applicationContext.xml` and modify the datasource properties `username`, `password` and `url` to use the credentials provided by MySQLs Add-on:

~~~xml
<property name="url" value="jdbc:mysql://${MYSQLS_HOSTNAME}:${MYSQLS_PORT}/${MYSQLS_DATABASE}"/>
<property name="username" value="${MYSQLS_USERNAME}"/>
<property name="password" value="${MYSQLS_PASSWORD}"/>
~~~

###Adjust Logger Configuration

Logging to a file is not recommended since the container's [file system] is not persistent. The default logger configuration - `src/main/resources/log4j.properties` should be modified to log either to `stdout/stderr` or `syslog`. Then cloudcontrol can  pick up all the messages and provide them to you via the [log command].

~~~xml
log4j.rootLogger=DEBUG, stdout
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=%p [%t] (%c) - %m%n
~~~

###Defining the Process Type

cloudControl uses the `Procfile` to start your application. Create the `Procfile` in your project root, pointing to the Jetty Runner:

~~~
web: java $JAVA_OPTS -jar target/dependency/jetty-runner.jar --port $PORT target/*.war
~~~

###Initializing Git Repository

Initialize a new git repository in the project directory and commit the files you have just created.

~~~bash
$ git init
$ git add pom.xml Procfile src
$ git commit -am "Initial commit"
~~~

##Pushing and Deploying your App


Choose a unique name (from now on called APP_NAME) for your application and create it on the cloudControl platform:

~~~bash
$ cctrlapp APP_NAME create java
~~~

Push your code to the application's repository:

~~~bash
$ cctrlapp APP_NAME/default push

-----> Receiving push
-----> Installing OpenJDK 1.6...done
-----> Installing settings.xml... done
-----> executing /srv/tmp/buildpack-cache/.maven/bin/mvn -B -Duser.home=/srv/tmp/builddir -Dmaven.repo.local=/srv/tmp/buildpack-cache/.m2/repository -s /srv/tmp/buildpack-cache/.m2/settings.xml -DskipTests=true clean install
       [INFO] Scanning for projects...
       [INFO]
       [INFO] ------------------------------------------------------------------------
       [INFO] Building APP_NAME 1.0-SNAPSHOT
       [INFO] ------------------------------------------------------------------------
       ...
       [INFO] Packaging webapp
       [INFO] Assembling webapp [petclinic] in [/srv/tmp/builddir/target/petclinic-0.1.0.BUILD-SNAPSHOT]
       [INFO] Processing war project
       [INFO] Copying webapp resources [/srv/tmp/builddir/src/main/webapp]
       [INFO] Webapp assembled in [365 msecs]
       [INFO] Building war: /srv/tmp/builddir/target/petclinic-0.1.0.BUILD-SNAPSHOT.war
       [INFO] WEB-INF/web.xml already added, skipping
       [INFO]
       [INFO] --- maven-dependency-plugin:2.3:copy (default) @ petclinic ---
       ...
       [INFO] ------------------------------------------------------------------------
       [INFO] BUILD SUCCESS
       [INFO] ------------------------------------------------------------------------
       [INFO] Total time: 3:38.174s
       [INFO] Finished at: Thu Juli 20 11:23:02 UTC 2013
       [INFO] Final Memory: 20M/229M
       [INFO] ------------------------------------------------------------------------
-----> Building image
-----> Uploading image (84M)

To ssh://APP_NAME@cloudcontrolled.com/repository.git
 * [new branch]      master -> master
~~~

Now you need the [MySQLs Add-on] to provide a MySQL database for your deployment. Check out the marketplace for [available plans] and then add one to your application:

~~~bash
$ cctrlapp APP_NAME/default addon.add mysqls.PLAN
~~~

Finally deploy your app:

~~~bash
$ cctrlapp APP_NAME/default deploy --max=6
~~~

The `--max=6` argument increases the container size to meet the high memory consumption of the Spring framework. Please note: increasing the size comes with additional costs.

Et voila, the app is now up and running at `http[s]://APP_NAME.cloudcontrolled.com` .


[Spring]: http://www.springsource.org/
[embedded Jetty server]: http://jetty.codehaus.org/Jetty/
[Jetty Runner]: http://wiki.eclipse.org/Jetty/Howto/Using_Jetty_Runner
[Spring Roo]: http://www.springsource.org/spring-roo
[MySQLs Add-on]: https://www.cloudcontrol.com/add-ons/mysqls
[cloudControl]: https://www.cloudcontrol.com/
[source code on Github]: https://github.com/cloudControl/java-spring-hibernate-example-app
[buildpack-java]: https://github.com/cloudControl/buildpack-java
[MySQLs Add-on]: https://www.cloudcontrol.com/dev-center/Add-on%20Documentation/Data%20Storage/MySQLs
[available plans]: https://www.cloudcontrol.com/add-ons/mysqls
[file system]: https://www.cloudcontrol.com/dev-center/Platform%20Documentation#non-persistent-filesystem
[Hibernate]: http://www.hibernate.org/
[MVC]: https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller
[log command]: https://www.cloudcontrol.com/dev-center/Platform%20Documentation#logging