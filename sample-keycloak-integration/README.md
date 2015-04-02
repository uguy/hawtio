Hawtio integration with Keycloak
================================

Those steps assume that you want your Hawt.io console to be secured by [Keycloak](http://www.keycloak.org) . Integration consists of 2 main steps.
First step is to download and prepare Keycloak server. Next step is deploy Hawt.io to your favourite server (JBoss Fuse, Karaf, Jetty, Tomcat, ...) and configure it
to use Keycloak for authentication.

Prepare Keycloak server
-----------------------

**1)** Download file [demorealm.json](demorealm.json) with Keycloak sample metadata about `hawtio-demo` realm. It's assumed you downloaded it to directory `/downloads` on your laptop. 

**2)**  Download keycloak appliance with wildfly from [http://sourceforge.net/projects/keycloak/files/1.1.0.Final/keycloak-appliance-dist-all-1.1.0.Final.zip/download](http://sourceforge.net/projects/keycloak/files/1.1.0.Final/keycloak-appliance-dist-all-1.1.0.Final.zip/download) . 
Then unpack and run keycloak server on localhost:8081 . You also need to import downloaded `demorealm.json` file into your Keycloak. Import can be done either via Keycloak admin console or by 
using `keycloak.import` system property:

```
unzip -q /downloads/keycloak-appliance-dist-all-1.1.0.Final.zip
cd keycloak-appliance-dist-all-1.1.0.Final/keycloak/bin/
./standalone.sh -Djboss.http.port=8081 -Dkeycloak.import=/downloads/demorealm.json
```

Realm has `hawtio-client` application installed as public client. There are couple of realm roles like `admin` and `viewer` . Names of these roles are same like 
default hawtio roles, which are allowed to login into hawtio admin console and to JMX.

There are also 3 users:

* **root** with password `password` and role `jmxAdmin`, so he is allowed to login into hawtio

* **john** with password `password` and role `viewer`, so he is allowed to login into hawtio

* **mary** with password `password` and no role assigned, so she is not allowed to login into hawtio


Hawtio and Keycloak integration on JBoss Fuse or Karaf
------------------------------------------------------

This was tested with JBoss Fuse 6.1.0-redhat379 and Apache Karaf 2.4 . Steps are almost same on both. Assuming `$FUSE_HOME` is the root directory of your fuse/karaf

* Add this into the end of file `$FUSE_HOME/etc/system.properties` :

``` 
hawtio.keycloakEnabled=true
hawtio.realm=keycloak
hawtio.keycloakClientConfig=${karaf.base}/etc/keycloak-hawtio-client.json
hawtio.rolePrincipalClasses=org.keycloak.adapters.jaas.RolePrincipal,org.apache.karaf.jaas.boot.principal.RolePrincipal
```

* Download and copy [keycloak-hawtio.json](keycloak-hawtio.json) and [keycloak-hawtio-client.json](keycloak-hawtio-client.json) into Fuse. 
File `keycloak-hawtio.json` is currently used for adapters on server (JAAS Login module) side. File `keycloak-hawtio-client.json` is used on client (Hawt.io JS application) side.
      
```
cp /downloads/keycloak-hawtio.json $FUSE_HOME/etc/
cp /downloads/keycloak-hawtio-client.json $FUSE_HOME/etc/
```
 
* Run Fuse or Karaf. 

```
cd $FUSE_HOME/bin
./fuse
```

Replace with `./karaf` if you are on plain Apache Karaf

* If you are on JBoss Fuse 6.1, you need to first uninstall old hawtio (This step is not needed on plain Apache karaf as it hasn't hawtio installed by default).
So in opened karaf terminal do this:

```
features:uninstall hawtio
features:uninstall hawtio-maven-indexer
features:uninstall hawtio-karaf-terminal
features:uninstall hawtio-core
features:removeurl mvn:io.hawt/hawtio-karaf/1.2-redhat-379/xml/features
```

* Install new hawtio with keycloak integration (Replace with the correct version where is Keycloak integration available. It should be 1.4.47 or newer) 

```
features:chooseurl hawtio 1.4.47
features:install hawtio
```

* Install keycloak OSGI bundling into Fuse/Karaf . It contains few jars with Keycloak adapter and also configuration of `keycloak` JAAS realm

```
features:addurl mvn:org.keycloak/keycloak-osgi-features/1.1.0.Final/xml/features
features:install keycloak-jaas
```

* Go to [http://localhost:8181/hawtio](http://localhost:8181/hawtio) and login in keycloak as `root` or `john` to see hawtio admin console. 
If you login as `mary`, you should receive 'forbidden' error in hawtio

#### Additional step on Karaf 2.4

From Karaf 2.4 there is more fine-grained security for JMX. Since Keycloak integration is currently using custom principal class `org.keycloak.adapters.jaas.RolePrincipal`
there is a need to add prefix with this class to the `etc/jmx.acl.*.cfg` files . Otherwise users root and john, who are logged via Keycloak, will be able to login
to Hawtio, but they won't have permission to do much here.  

This is likely going to be improved in the future, however currently 
you may need to edit this in file `$KARAF_HOME/etc/jmx.acl.cfg` (and maybe also other `jmx.acl.*.cfg` files according to permission you want):

```
list* = org.keycloak.adapters.jaas.RolePrincipal:viewer
get* = org.keycloak.adapters.jaas.RolePrincipal:viewer
is* = org.keycloak.adapters.jaas.RolePrincipal:viewer
set* = org.keycloak.adapters.jaas.RolePrincipal:admin
* = org.keycloak.adapters.jaas.RolePrincipal:admin
```  


Hawtio and Keycloak integration on Jetty 
----------------------------------------


Assuming you deployed Hawtio WAR to Jetty as described in [http://hawt.io/getstarted/index.html](http://hawt.io/getstarted/index.html) . 
Integration was tested with Jetty 8.1.16 .

* Create file `JETTY_HOME/etc/login.conf` with the content like this:
 
```
hawtio {
  org.keycloak.adapters.jaas.BearerTokenLoginModule required 
  keycloak-config-file="${hawtio.keycloakServerConfig}"; 
};
```
 
* Download and copy [keycloak-hawtio.json](keycloak-hawtio.json) and [keycloak-hawtio-client.json](keycloak-hawtio-client.json) into Jetty. 
  File `keycloak-hawtio.json` is currently used for adapters on server (JAAS Login module) side. File `keycloak-hawtio-client.json` is used on client (Hawt.io JS application) side.
   
```
cp /downloads/keycloak-hawtio.json $JETTY_HOME/etc/
cp /downloads/keycloak-hawtio-client.json $JETTY_HOME/etc/
```

* Install Keycloak jetty adapter into your Jetty server as described on [http://docs.jboss.org/keycloak/docs/1.1.0.Final/userguide/html/ch08.html#jetty8-adapter](http://docs.jboss.org/keycloak/docs/1.1.0.Final/userguide/html/ch08.html#jetty8-adapter).
 
* Export JETTY_HOME in your terminal. For example: 

```
export $JETTY_HOME=/mydir/jetty-distribution-8.1.16.v20140903
```

* Export JAVA_OPTIONS and add all necessary system options similarly like this:

```
export JAVA_OPTIONS="-Dhawtio.authenticationEnabled=true -Dhawtio.realm=hawtio -Dhawtio.keycloakEnabled=true -Dhawtio.roles=admin,viewer -Dhawtio.rolePrincipalClasses=org.keycloak.adapters.jaas.RolePrincipal 
-Dhawtio.keycloakClientConfig=$JETTY_HOME/etc/keycloak-hawtio-client.json -Dhawtio.keycloakServerConfig=$JETTY_HOME/etc/keycloak-hawtio.json 
-Djava.security.auth.login.config=$JETTY_HOME/etc/login.conf"
```

* Run Jetty and go to [http://localhost:8080/hawtio](http://localhost:8080/hawtio) . Users are again `root` and `john` with access and `mary` without access.


Hawtio and Keycloak integration on Tomcat
-----------------------------------------

Instructions are quite similar to Jetty, you would need to setup JAAS realm and set the system properties. Just use Tomcat adapter instead of the Jetty one.
Also you may need to add system property `-Dhawtio.authenticationContainerDiscoveryClasses=` (really empty value). This is needed, so that
Tomcat will use configured JAAS realm with BearerTokenLoginModule instead of `tomcat-users.xml` file, which Hawtio uses on Tomcat by default.
 

Hawtio and Keycloak integration on Wildfly
------------------------------------------

This is even easier as you can use same WildFly server where Keycloak is already running. No need to have separate server, but you can use separate server if you want. 

So in next steps we will use the existing Keycloak server on localhost:8081 and assume that Hawtio WAR is already deployed on WildFly as 
described in [http://hawt.io/getstarted/index.html](http://hawt.io/getstarted/index.html) .

* Download and copy [keycloak-hawtio.json](keycloak-hawtio.json) and [keycloak-hawtio-client.json](keycloak-hawtio-client.json) into Wildfly. 
  File `keycloak-hawtio.json` is currently used for adapters on server (JAAS Login module) side. File `keycloak-hawtio-client.json` is used on client (Hawt.io JS application) side.
   
```
cp /downloads/keycloak-hawtio.json $JBOSS_HOME/standalone/configuration/
cp /downloads/keycloak-hawtio-client.json $JBOSS_HOME/standalone/configuration/
```

* In `$JBOSS_HOME/standalone/configuration/standalone.xml` configure system properties like this:

```
<extensions>
...
</extensions>

<system-properties>
    <property name="hawtio.authenticationEnabled" value="true" />
    <property name="hawtio.realm" value="hawtio" />
    <property name="hawtio.roles" value="admin,viewer" />
    <property name="hawtio.rolePrincipalClasses" value="org.keycloak.adapters.jaas.RolePrincipal" />
    <property name="hawtio.keycloakEnabled" value="true" />
    <property name="hawtio.keycloakClientConfig" value="${jboss.server.config.dir}/keycloak-hawtio-client.json" />
    <property name="hawtio.keycloakServerConfig" value="${jboss.server.config.dir}/keycloak-hawtio.json" />
</system-properties>
```

Also add hawtio realm to this file to `security-domains` section:

```
<security-domain name="hawtio" cache-type="default">
    <authentication>
        <login-module code="org.keycloak.adapters.jaas.BearerTokenLoginModule" flag="required">
            <module-option name="keycloak-config-file" value="${hawtio.keycloakServerConfig}"/>
        </login-module>                        
    </authentication>
</security-domain>
```

* Run WildFly on port 8081 as described in [Prepare Keycloak Server](#prepare-keycloak-server) section and go to [http://localhost:8081/hawtio](http://localhost:8081/hawtio) . 
Users are again `root` and `john` with access and `mary` without access. 

