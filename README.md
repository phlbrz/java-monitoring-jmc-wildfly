# java-monitoring-jmc-wildfly
How to monitor an application in wildfly with jmc

## Objective
In my search for a solution, there are numerous options available. However, my intention is to consolidate main part in one place.  

- How to live monitor an application in Wildfly using JMC - JDK Mission Control.
- Application: Java 8 latest, Widlfly 18.0.1.Final or newer.
- Developer machine: you can use Java 11 or newer, jmc.
- This is for a remote connection with jmx, it is going to work if you're using Wildfly application in a locally docker container or a remote Wildfly application.
- I never tested a local Wildfly without container, so it keeps my local environment clean, but I think it should work also.

## Gratitude
I would like to express my sincere gratitude to all those who contribute to this work and strive to provide users with an exceptional experience.  
Your dedication and efforts in delivering excellence are truly commendable.

Thank you for your valuable contributions and for making a difference in the way users interact with our products/services.  
Together, we can continue to create a seamless and remarkable user experience.

## Information about this repository

Repository structure:

All in this repository you can find in the references or searching through the web.

- You don't need to download Wildfly, but if you preffer, here is a direct link: [Wildfly 18.0.1.Final](https://download.jboss.org/wildfly/18.0.1.Final/wildfly-18.0.1.Final.zip)
- You'll need to download JMC, so here a direct link: [jmc-8.3.1_linux-x64.tar.gz](https://download.java.net/java/GA/jmc8/05/binaries/jmc-8.3.1_linux-x64.tar.gz)

The files that are ready in this repository:

- `jboss-client.jar` - from Wildfly 18.0.1.Final and patched from <https://github.com/jiekang/sandbox/tree/EAPSUP-143/extension-poc>, solving GSSManager connection error with JMX, Wildfly, JMC.
- `jboss-cli-client.jar` - from Wildfly 18.0.1.Final.
- `jmc.ini` - jmc ini config to help you to configure the xbootclasspath.

## Developer local folder structure

It's a suggestion to help you to clone the repository and extract jmc.

- Create a directory for your work.

```shell
mkdir -p $HOME/workspace-monitoring/ 
cd $HOME/workspace-monitoring/
```

- Download JMC and extract:

```shell
mkdir -p $HOME/workspace-monitoring/jmc
cd $HOME/workspace-monitoring/jmc/
curl -O https://download.java.net/java/GA/jmc8/05/binaries/jmc-8.3.1_linux-x64.tar.gz
tar zxvf jmc-8.3.1_linux-x64.tar.gz # it creates a folder jmc-8.3.1_linux-x64
cd jmc-8.3.1_linux-x64
pwd 
> $HOME/workspace-monitoring/jmc/jmc-8.3.1_linux-x64/
```

## How to configure in Wildfly

I like and I use G1GC in my applications, in this example I use a low memory just to a small application.  

`JAVA_OPTS="$JAVA_OPTS -XX:+UseG1GC -XX:+UseStringDeduplication -XX:+UseCompressedOops -Xms512m -Xmx2048m -XX:MaxMetaspaceSize=256m"`

Also, this applications use a Java version 1.8.0 from a centos distro, you can found it in `FROM jboss/wildfly:18.0.1.Final`.  
To keep this image updated, I recommend to update, see this snippet:

```dockerfile
USER root

RUN yum -y remove java-11-* \
  && yum -y install java-1.8.0-openjdk-devel \
  && yum -y update \
  && yum -y autoremove \
  && yum clean all \
  && rm -rf /var/cache/yum
ENV JAVA_HOME /usr/lib/jvm/java-1.8.0-openjdk
```

In the Dockerfile has a RUN instruction that runs the `add-user.sh` script to add an user and password to jmx.

```Dockerfile
RUN /opt/jboss/wildfly/bin/add-user.sh adminUsr adminPasswdEasy
```

And I tell to Dockerfile to expose the ports:
```Dockerfile
EXPOSE 8080 9990
```

I don't expose the port 9990 to the world, so I can keep user and password simple.

Also, in CMD instruction, I tell the wildfly to expose the management:

```Dockerfile
CMD ["/opt/jboss/wildfly/bin/standalone.sh", "-c", "standalone.xml", "-b", "0.0.0.0", "-bmanagement", "0.0.0.0"]
```

To test, deploy your war, build the dockerfile and run locally using docker:

```shell
docker build . -t java-monitoring-application-jmx-jmc-wildfly
docker run -e JAVA_OPTS="-XX:+UseG1GC -XX:+UseStringDeduplication -XX:+UseCompressedOops -Xms128m -Xmx2g -XX:MaxMetaspaceSize=384m -XX:NativeMemoryTracking=detail -Djava.net.preferIPv4Stack=true -Djboss.modules.system.pkgs=$JBOSS_MODULES_SYSTEM_PKGS -Djava.awt.headless=true" -p 8080:8080 -p 9990:9990 -it java-monitoring-application-jmx-jmc-wildfly
```

## How to expose the port in Kubernetes or Openshift

It's similar in both cases, so I'm going to give an example after each link instruction that I don't cover at the moment.

You can find instructions to forward port in Kubernetes here:  
- Any version: <https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/#forward-a-local-port-to-a-port-on-the-pod>

```shell
kubectl port-forward mypod 9990:9990
```

You can find instructions to forward port in Openshift here:  
- 4.x: https://docs.openshift.com/container-platform/4.13/nodes/containers/nodes-containers-port-forwarding.html
- 3.x: https://docs.openshift.com/container-platform/3.11/dev_guide/port_forwarding.html

```shell
oc port-forward mypod 9990:9990
```

## How to configure and use JMC

- I recommend cloning this repository, this will provide you with the fastest way to get started.

```shell
cd $HOME/workspace-monitoring/ 
git clone https://github.com/phlbrz/java-monitoring-jmc-wildfly.git
cd java-monitoring-jmc-wildfly
```

- Copy the files below to jmc folder: 

```shell
cp jboss-client.jar $HOME/workspace-monitoring/jmc/jmc-8.3.1_linux-x64/JDK\ Mission\ Control/dropins/
cp jmc.ini $HOME/workspace-monitoring/jmc/jmc-8.3.1_linux-x64/JDK\ Mission\ Control/
```

- If you don't follow this instructions, you'll need to modify the `-Xbootclasspath/a` line in `$HOME/workspace-monitoring/jmc/jmc-8.3.1_linux-x64/JDK\ Mission\ Control/jmc.ini`, because this line has to be the path to `jboss-cli-client.jar`.
- In this repository, it points to `-Xbootclasspath/a:$HOME/workspace-monitoring/java-monitoring-jmc-wildfly/jboss-cli-client.jar`.
- Modify your properties in `jmc.ini`, like `Xmx`, if necessary.

- Run jmc

```shell
./jmc.sh
```

- In JDK Mission Control, go to File -> Connect -> Create a new connection
- Click on Custom JMX service URL
- Replace the string JMX service URL: `service:jmx:rmi:///jndi/rmi://localhost:7091/jmxrmi` with `service:jmx:remote+http://localhost:9990` (or any port that you exposed the wildfly)
- Fill the user and password as you configured in `add-user.sh` command.
- You can store the credentials in settings file, just mark and set a master password (I'm using Gnome).
- Click on Test connection, if you already exposed the port in wildfly, in the container or through kubernetes/openshift, it's going change the status from Untested to OK.
- Click Finish
- Expand the localhost:9990 connection created
- Two clicks at the MBean Server and wait, it takes some time to connect and then shows the Overview tab.

![image](https://github.com/phlbrz/java-monitoring-jmc-wildfly/assets/6169365/aec89237-7386-4c41-9b16-3f94ecac87dc)

## Sources and references

<https://www.google.com/search?q=jmx+wildfly+jmc>  
<https://jdk.java.net/jmc/8/>  
<https://docs.wildfly.org/18/Admin_Guide.html>  
<https://stackoverflow.com/questions/29432354/how-to-open-wildfly-8-2-jmx-port-for-monitoring>  
<https://docs.tibco.com/pub/mdm/9.1.0/doc/html/GUID-7E2C2C30-50EF-4903-B4FA-EEDF6F457249.html>  
<https://github.com/jiekang/sandbox/tree/EAPSUP-143/extension-poc>  
<https://bugzilla.redhat.com/show_bug.cgi?id=1933126>  
<https://access.redhat.com/solutions/5897561>  
<https://access.redhat.com/solutions/700283>
