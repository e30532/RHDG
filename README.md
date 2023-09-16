This document describes a simple use case of Red Hat Data Grid with liberty.

1. deploy OCP4.13.4 (3 master, 3 worker)    
2. login to the ocp console and deploy Data Grid operator     
<img width="300" alt="image" src="https://github.com/e30532/RHDG/assets/22098113/63ce715b-5d54-455a-a57f-691d97ce855c"><br>
<img width="300" alt="image" src="https://github.com/e30532/RHDG/assets/22098113/671b01c3-e033-4968-88b7-8f1842cf0b93"><br>
<img width="300" alt="image" src="https://github.com/e30532/RHDG/assets/22098113/380fa611-8807-4160-aa32-b57850009012"><br>  
<img width="300" alt="image" src="https://github.com/e30532/RHDG/assets/22098113/5b59ca67-e3aa-4c8d-b625-9885fd8d6e0b"><br>
3. login to the NFS server which can be accessed from OCP nodes. (It's an infrastracture node in my env.)   
   ```
   ***@***-MacBook-Pro ~ % ping api.*.*.*.*.com
   ***@***-MacBook-Pro ~ % sudo vi /private/etc/hosts
   9.*.*.* cp4a api.*.*.*.*.com oauth-openshift.apps.*.*.*.*.com
   ***@***-MacBook-Pro ~ % ssh-copy-id root@cp4a
   ***@***-MacBook-Pro ~ % ssh root@cp4a
   ```  
4. create a NFS storage class   
    4.1. On NFS server, create a NFS directory and expose it to the 6 OCP nodes.
    ```
    # mkdir -p /data/mynfs1
    # vi /etc/exports
    # exportfs -ra
    # exportfs -v
    /data/mynfs1  	10.*.*.*(sync,no_wdelay,hide,no_subtree_check,sec=sys,rw,insecure,no_root_squash,no_all_squash)
    :
    :
    /data/mynfs1  	10.*.*.*(sync,no_wdelay,hide,no_subtree_check,sec=sys,rw,insecure,no_root_squash,no_all_squash)
    ```
    4.2 confirm the IP address of the NFS server.
    ```
    # ip addr | egrep '10\.'
    inet 10.*.*.*/19 brd 10.*.*.255 scope global dynamic noprefixroute ens3
    ```
    4.3. Download [storage-class.yml](https://raw.githubusercontent.com/e30532/RHDG/main/storage-class.yml), [persitent-volumne-claim.yml](https://raw.githubusercontent.com/e30532/RHDG/main/persitent-volumne-claim.yml), [rbac.yml](https://raw.githubusercontent.com/e30532/RHDG/main/rbac.yml) and [deployment.yml](https://raw.githubusercontent.com/e30532/RHDG/main/deployment.yml)   
    4.4. Update two points in deployment.xml with the NFS server IP address   
    4.5. login to the OCP cluster and create the resources which are related to NFS dynamic storage provisioning
    ```
    ***@***-MacBook-Pro ~ % oc login https://oauth-openshift.apps.*.*.*.*.com:6443 -u kubeadmin -p *********
    ***@***-MacBook-Pro ~ % oc apply -f storage-class.yml -f persitent-volumne-claim.yml -f rbac.yml -f deployment.yml
    ```
    4.6. make the storage class as default
    ```
    ***@***-MacBook-Pro ~ % oc patch storageclass nfs -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "true"}}}'
    storageclass.storage.k8s.io/nfs patched
    ***@***-MacBook-Pro ~ % oc get sc
    NAME            PROVISIONER       RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
    nfs (default)   lab.hoge.jp/nfs   Retain          Immediate           false                  84s
    ```
5. create a infinispan cluster   
<img width="300" alt="image" src="https://github.com/e30532/RHDG/assets/22098113/1f09e2a4-2963-47af-b89e-805cfd0de69f"><br>
<img width="300" alt="image" src="https://github.com/e30532/RHDG/assets/22098113/5643f450-8cb0-436f-94d3-85adba63635d"><br>
expose the service to route   
<img width="300" alt="image" src="https://github.com/e30532/RHDG/assets/22098113/9e3a73f0-3909-4934-8b24-f8940a41d776"><br>
enable authentication   
<img width="300" alt="image" src="https://github.com/e30532/RHDG/assets/22098113/dc9da8d3-2829-4ce0-a666-ecc6da8fde70"><br>
6. confirm the status
```
% oc project e30532
% oc get infinispan
NAME                 AGE
example-infinispan   72s
% oc get sts
NAME                 READY   AGE
example-infinispan   1/1     79s
% oc get pod
NAME                                                 READY   STATUS    RESTARTS   AGE
example-infinispan-0                                 1/1     Running   0          82s
example-infinispan-config-listener-79b9fd76f-qfq4m   1/1     Running   0          52s
% oc get route
NAME                          HOST/PORT                                                         PATH   SERVICES             PORT    TERMINATION   WILDCARD
example-infinispan-external   example-infinispan-external-e30532.apps.*.*.*.*.com          example-infinispan   11222   passthrough   None 
```
7. login to data grid console   
   https://example-infinispan-external-e30532.apps.*.*.*.*.com   
   <img width="300" alt="image" src="https://github.com/e30532/RHDG/assets/22098113/c91609cc-33af-46dc-a022-30a1f63c12f8"><br>
   The password can be confirmed in the following secret.
   ```
   % oc get infinispan -o yaml
   :
         endpointSecretName: example-infinispan-generated-secret
   :
   % oc get secret example-infinispan-generated-secret -o jsonpath="{.data.identities\.yaml}" | base64 --decode
   ```
   <img width="300" alt="image" src="https://github.com/e30532/RHDG/assets/22098113/c3090bd7-fbcb-4ae8-8817-4b8f7c7bb554"><br>

8. Here is an example how we access the grid from external java cache client.
   First, create a cache in a grid.   
   <img width="300" alt="image" src="https://github.com/e30532/RHDG/assets/22098113/18a9d41c-7e37-4c7e-8202-8d44c8b23902"><br>
   <img width="300" alt="image" src="https://github.com/e30532/RHDG/assets/22098113/93f75d79-ef14-4841-ac54-4a16dd0d52ce"><br>
   <img width="300" alt="image" src="https://github.com/e30532/RHDG/assets/22098113/073bffd7-f117-48da-a096-d1c4209df7ac"><br>
   <img width="300" alt="image" src="https://github.com/e30532/RHDG/assets/22098113/caf94d33-c07f-48ff-ba0d-55e5d0555852"><br>

   At the remote side, create a java project and convert it to a Maven project. And add the dependency.    
   <img width="300" alt="image" src="https://github.com/e30532/RHDG/assets/22098113/75d37b9f-2418-49d7-8a11-55513a96be63"><br>
   <img width="300" alt="image" src="https://github.com/e30532/RHDG/assets/22098113/574393fa-1d95-471e-b360-b89b3b54ff8c"><br>
   <img width="300" alt="image" src="https://github.com/e30532/RHDG/assets/22098113/c5c3710b-25c4-4bff-b20c-1b584ce7a875"><br>
   ```
   <dependencies>
  	   <dependency>
  		   <groupId>org.infinispan</groupId>
  		   <artifactId>infinispan-client-hotrod</artifactId>
  		   <version>14.0.17.Final</version>
     </dependency>
   </dependencies>
   ```
   <img width="300" alt="image" src="https://github.com/e30532/RHDG/assets/22098113/eecf7882-c54f-4d2c-b8e1-efc4e9794b81"><br>
   <img width="300" alt="image" src="https://github.com/e30532/RHDG/assets/22098113/17bc0509-a064-4033-8df2-5d23f94cb633"><br>
   <img width="300" alt="image" src="https://github.com/e30532/RHDG/assets/22098113/eedceed0-c5d5-4932-a300-4ad5707ad81f"><br>
   ```
   package test;

   import org.infinispan.client.hotrod.RemoteCacheManager;
   import org.infinispan.client.hotrod.configuration.ClientIntelligence;
   import org.infinispan.client.hotrod.configuration.ConfigurationBuilder;
   public class MyMain {
      public static void main(String[] args) {
         // TODO Auto-generated method stub
         ConfigurationBuilder builder = new ConfigurationBuilder();
         builder.addServer()
		         .host("example-infinispan-external-e30532.apps.*.*.*.*.com")
		         .port(443)
		         .security().authentication()
		         .username("developer")
		         .password("*******")
		         .realm("default")
		         .saslMechanism("PLAIN")
		         .ssl()
		         .sniHostName("example-infinispan-external-e30532.apps.*.*.*.*.com")
		         .trustStoreFileName("/root/trust.p12")
		         .trustStorePassword("WebAS".toCharArray())
	             .trustStoreType("PKCS12");
   		builder.clientIntelligence(ClientIntelligence.BASIC);
	   	RemoteCacheManager cacheManager = new RemoteCacheManager(builder.build());
		   cacheManager.getCache("testCache").put("1", "Chihiro");
		   System.out.println(cacheManager.getCache("testCache").get("1"));
		}
   }
   ```
   To establish a ssl connection, the client side trust store needs to have the DG certificate.
   ```
   % oc get infinispan -o yaml
   :
       security:
      endpointAuthentication: true
      endpointEncryption:
        certSecretName: example-infinispan-cert-secret
   :

   % oc get secret example-infinispan-cert-secret -o jsonpath="{.data.tls\.crt}" | base64 --decode > trust.crt
   % scp trust.crt root@fyre1:/root/

   # /opt/IBM/HTTPServer90/bin/gskcapicmd -keydb -create -db /root/trust.p12 -pw WebAS -type pkcs12 -stash
   # /opt/IBM/HTTPServer90/bin/gskcapicmd -cert -add -db /root/trust.p12 -pw WebAS -label dg -file /root/trust.crt
   ```
   <img width="300" alt="image" src="https://github.com/e30532/RHDG/assets/22098113/e92d32aa-7a50-47bc-9c12-f3d3d7c4c178">

9. Finally here is an example to use DG as a http session store.
```
# pwd
/root/skill/liberty
# cat pom.xml 
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
   <groupId>io.openliberty</groupId>
  <artifactId>openliberty-infinispan-client</artifactId>
  <version>1.0</version>
  <dependencies>
    <dependency>
      <groupId>org.infinispan</groupId>
      <artifactId>infinispan-jcache-remote</artifactId>
      <version>14.0.17.Final</version>
    </dependency>
  </dependencies>
</project>
# mvn -f ./pom.xml versions:use-latest-releases -DallowMajorUpdates=false
# mvn -f ./pom.xml dependency:copy-dependencies -DoutputDirectory=./lib/
# cat Dockerfile 
FROM websphere-liberty:23.0.0.8-full-java17-openj9
MAINTAINER Yoshiki Yamada, e30532@jp.ibm.com
COPY --chown=1001:0  SimpleSession.ear /config/dropins/SimpleSession.ear
COPY --chown=1001:0  server.xml /config/server.xml
COPY --chown=1001:0  trust.p12 /config/trust.p12
COPY --chown=1001:0  lib /liberty/usr/shared/resources/
ARG VERBOSE=true
ENV WLP_LOGGING_CONSOLE_FORMAT=JSON
ENV WLP_LOGGING_CONSOLE_LOGLEVEL=info
ENV WLP_LOGGING_CONSOLE_SOURCE=message,trace,accessLog,ffdc,audit
RUN configure.sh
# cat server.xml 
<?xml version="1.0" encoding="UTF-8"?>
<server description="new server">

    <!-- Enable features -->
    <featureManager>
        <feature>jsp-2.3</feature>
        <feature>javaee-7.0</feature>
        <feature>sessionCache-1.0</feature>
    </featureManager>
    <keyStore id="defaultKeyStore" password="passw0rd" />
    <!-- To access this server from a remote client add a host attribute to the following element, e.g. host="*" -->
    <httpEndpoint id="defaultHttpEndpoint"
                  host="*"
                  httpPort="9080"
                  httpsPort="9443" />
                  
    <!-- Automatically expand WAR files and EAR files -->
    <applicationManager autoExpand="true"/>

    <httpSessionCache cacheManagerRef="CacheManager"/>
    <library id="InfinispanLib">
        <fileset dir="/liberty/usr/shared/resources/" includes="*.jar"/>
    </library>
    <cacheManager id="CacheManager">
        <properties
            infinispan.client.hotrod.server_list="example-infinispan"
            infinispan.client.hotrod.auth_username="developer"
            infinispan.client.hotrod.auth_password="***********"
            infinispan.client.hotrod.auth_realm="default"
            infinispan.client.hotrod.use_ssl="true"
            infinispan.client.hotrod.trust_store_file_name="/config/trust.p12"
            infinispan.client.hotrod.trust_store_password="WebAS"
            infinispan.client.hotrod.trust_store_type="PKCS12"
            infinispan.client.hotrod.sni_host_name="example-infinispan.e30532.svc"
            infinispan.client.hotrod.sasl_mechanism="PLAIN"
            infinispan.client.hotrod.java_serial_whitelist=".*"
            infinispan.client.hotrod.marshaller="org.infinispan.commons.marshall.JavaSerializationMarshaller"/>
        <cachingProvider jCacheLibraryRef="InfinispanLib" />
    </cacheManager>
</server>
# cp /root/trust.p12 ./
# cp /root/SimpleApps/SimpleSession.ear ./
# ls -l
total 24
-rw-r--r-- 1 root root  499 Sep 16 00:22 Dockerfile
-rw-r--r-- 1 root root 3885 Sep 15 21:19 SimpleSession.ear
drwxr-xr-x 2 root root 4096 Sep 16 00:22 lib
-rw-r--r-- 1 root root  577 Sep 16 00:21 pom.xml
-rw-r----- 1 root root 1866 Sep 15 21:58 server.xml
-rw------- 1 root root 2970 Sep 16 00:26 trust.p12
# podman build -t simplesession .

export the default image registry.
% oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge
config.imageregistry.operator.openshift.io/cluster patched
% oc get route -n openshift-image-registry
NAME            HOST/PORT                                                             PATH   SERVICES         PORT    TERMINATION   WILDCARD
default-route   default-route-openshift-image-registry.apps.*.*.*.*.com          image-registry   <all>   reencrypt     None
yamayoshi@yamadayoshikis-MacBook-Pro ~ % oc whoami -t
sha256~1*I
yamayoshi@yamadayoshikis-MacBook-Pro ~ %

# podman tag simplesession:latest 'default-route-openshift-image-registry.apps.*.*.*.*.com/e30532/simplesession'
# vi /etc/containers/registries.conf
:
[[registry]]
location = "default-route-openshift-image-registry.apps.*.*.*.*.com"
insecure = true
:

# podman login -u kubeadmin -p sha256~1****I https://default-route-openshift-image-registry.apps.*.*.*.*.com
# podman push default-route-openshift-image-registry.apps.*.*.*.*.com/e30532/simplesession


% oc get is
NAME            IMAGE REPOSITORY                                                                           TAGS     UPDATED
simplesession   default-route-openshift-image-registry.apps.*.*.*.*.com/e30532/simplesession   latest   3 minutes ago
% oc new-app simplesession:latest
warning: Cannot check if git requires authentication.
--> Found image cab3301 (9 minutes old) in image stream "e30532/simplesession" under tag "latest" for "simplesession:latest"


--> Creating resources ...
    deployment.apps "simplesession" created
    service "simplesession" created
--> Success
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose service/simplesession' 
    Run 'oc status' to view your app.
% oc expose service/simplesession
route.route.openshift.io/simplesession exposed
% oc get route/simplesession
NAME            HOST/PORT                                           PATH   SERVICES        PORT       TERMINATION   WILDCARD
simplesession   simplesession-e30532.apps.*.*.*.*.com          simplesession   9080-tcp                 None
% 
yamayoshi@yamadayoshikis-MacBook-Pro ~ % oc get pod                                  
NAME                                                 READY   STATUS    RESTARTS   AGE
example-infinispan-0                                 1/1     Running   0          88m
example-infinispan-config-listener-79b9fd76f-qfq4m   1/1     Running   0          88m
simplesession-865dc67844-wnw4d                       1/1     Running   0          8s
% oc exec -it simplesession-865dc67844-wnw4d  -- cat /logs/messages.log
********************************************************************************
product = WebSphere Application Server 23.0.0.8 (wlp-1.0.80.cl230820230807-0401)
wlp.install.dir = /opt/ibm/wlp/
server.output.dir = /opt/ibm/wlp/output/defaultServer/
java.home = /opt/java/openjdk
java.version = 17.0.8
java.runtime = IBM Semeru Runtime Open Edition (17.0.8+7)
os = Linux (5.14.0-284.18.1.el9_2.x86_64; amd64) (en_US)
process = 1@simplesession-865dc67844-wnw4d
Classpath = /opt/ibm/wlp/bin/tools/ws-server.jar
Java Library path = /opt/java/openjdk/lib/default:/opt/java/openjdk/lib:/usr/lib64:/usr/lib
********************************************************************************
[9/16/23, 7:48:43:400 UTC] 00000001 com.ibm.ws.kernel.launch.internal.FrameworkManager           A CWWKE0001I: The server defaultServer has been launched.
[9/16/23, 7:48:43:411 UTC] 00000001 com.ibm.ws.kernel.launch.internal.FrameworkManager           A CWWKE0100I: This product is licensed for development, and limited production use. The full license terms can be viewed here: https://public.dhe.ibm.com/ibmdl/export/pub/software/websphere/wasdev/license/base_ilan/ilan/23.0.0.8/lafiles/en.html
[9/16/23, 7:48:45:438 UTC] 00000031 com.ibm.ws.config.xml.internal.ServerXMLConfiguration        A CWWKG0093A: Processing configuration drop-ins resource: /opt/ibm/wlp/usr/servers/defaultServer/configDropins/defaults/keystore.xml
[9/16/23, 7:48:45:554 UTC] 00000001 com.ibm.ws.kernel.launch.internal.FrameworkManager           I CWWKE0002I: The kernel started after 2.388 seconds
[9/16/23, 7:48:45:587 UTC] 0000003a com.ibm.ws.kernel.feature.internal.FeatureManager            I CWWKF0007I: Feature update started.
[9/16/23, 7:48:45:652 UTC] 0000003a com.ibm.ws.config.xml.internal.ConfigValidator               A CWWKG0102I: Found conflicting settings for defaultKeyStore instance of keyStore configuration.
  Property password has conflicting values:
    Secure value is set in file:/opt/ibm/wlp/usr/servers/defaultServer/configDropins/defaults/keystore.xml.
    Secure value is set in file:/opt/ibm/wlp/usr/servers/defaultServer/server.xml.
  Property password will be set to the value defined in file:/opt/ibm/wlp/usr/servers/defaultServer/server.xml.

[9/16/23, 7:48:45:712 UTC] 0000002d com.ibm.ws.security.ready.internal.SecurityReadyServiceImpl  I CWWKS0007I: The security service is starting...
[9/16/23, 7:48:45:815 UTC] 0000002f com.ibm.ws.app.manager.internal.monitor.DropinMonitor        A CWWKZ0058I: Monitoring dropins for applications.
[9/16/23, 7:48:45:978 UTC] 0000002e com.ibm.ws.cache.ServerCache                                 I DYNA1001I: WebSphere Dynamic Cache instance named baseCache initialized successfully.
[9/16/23, 7:48:45:979 UTC] 0000002e com.ibm.ws.cache.ServerCache                                 I DYNA1071I: The cache provider default is being used.
[9/16/23, 7:48:45:986 UTC] 0000002e com.ibm.ws.cache.CacheServiceImpl                            I DYNA1056I: Dynamic Cache (object cache) initialized successfully.
[9/16/23, 7:48:46:088 UTC] 00000043 com.ibm.ws.security.token.ltpa.internal.LTPAKeyCreateTask    I CWWKS4105I: LTPA configuration is ready after 0.007 seconds.
[9/16/23, 7:48:46:191 UTC] 00000036 com.ibm.ws.ssl.config.WSKeyStore                             I Successfully loaded default keystore: /opt/ibm/wlp/output/defaultServer/resources/security/key.p12 of type: PKCS12
[9/16/23, 7:48:46:240 UTC] 0000002d ibm.ws.security.authentication.internal.jaas.JAASServiceImpl I CWWKS1123I: The collective authentication plugin with class name NullCollectiveAuthenticationPlugin has been activated. 
[9/16/23, 7:48:46:315 UTC] 0000002d com.ibm.ws.security.jaspi.AuthConfigFactoryWrapper           I CWWKS1655I: The default Java Authentication SPI for Containers (JASPIC) AuthConfigFactory class com.ibm.ws.security.jaspi.ProviderRegistry is being used because the Java security property authconfigprovider.factory is not set. 
[9/16/23, 7:48:46:356 UTC] 00000048 org.infinispan.CONFIG                                        W ISPN000959: Property 'infinispan.client.hotrod.java_serial_whitelist' has been deprecated. Please use 'infinispan.client.hotrod.java_serial_allowlist' instead.
[9/16/23, 7:48:46:594 UTC] 00000048 SystemErr                                                    R SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
[9/16/23, 7:48:46:595 UTC] 00000048 SystemErr                                                    R SLF4J: Defaulting to no-operation (NOP) logger implementation
[9/16/23, 7:48:46:595 UTC] 00000048 SystemErr                                                    R SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
[9/16/23, 7:48:46:644 UTC] 00000048 org.infinispan.HOTROD                                        I ISPN004108: Native IOUring transport not available, using NIO instead: io.netty.incubator.channel.uring.IOUring
[9/16/23, 7:48:46:834 UTC] 00000058 org.infinispan.SECURITY                                      I ISPN000947: Using Java SSL Provider
[9/16/23, 7:48:46:947 UTC] 0000002d com.ibm.ws.sib.utils.ras.SibMessage                          I  CWSIS1553I: The minimum reserved size in the configuration information of the permanent store file is 20,971,520 bytes. The maximum size is 209,715,200 bytes.
[9/16/23, 7:48:46:948 UTC] 0000002d com.ibm.ws.sib.utils.ras.SibMessage                          I  CWSIS1557I: The minimum reserved size in the configuration information of the temporary store file is 20,971,520 bytes. The maximum size is 209,715,200 bytes.
[9/16/23, 7:48:47:163 UTC] 00000048 org.infinispan.HOTROD                                        I ISPN004021: Infinispan version: Infinispan 'Flying Saucer' 14.0.17.Final
[9/16/23, 7:48:47:265 UTC] 0000002d com.ibm.ws.sib.utils.ras.SibMessage                          I  CWSID0108I: JMS server has started.  
[9/16/23, 7:48:47:498 UTC] 0000002e .apache.cxf.cxf.core.3.2:1.0.80.cl230820230807-0401(id=215)] I Aries Blueprint packages not available. So namespaces will not be registered
[9/16/23, 7:48:47:560 UTC] 00000036 org.apache.cxf.ext.logging.osgi.Activator                    I CXF message logging feature disabled
[9/16/23, 7:48:47:753 UTC] 0000006b com.ibm.ws.app.manager.AppMessageHelper                      I CWWKZ0018I: Starting application SimpleSession.
[9/16/23, 7:48:47:923 UTC] 0000006b org.jboss.weld.Version                                       I WELD-000900: 2.4.8 (Final)
[9/16/23, 7:48:48:039 UTC] 0000006b com.ibm.ws.session.WASSessionCore                            I SESN8502I: The session manager found a persistent storage location; it will use session persistence mode=JCACHE
[9/16/23, 7:48:48:046 UTC] 0000006b com.ibm.ws.webcontainer.osgi.webapp.WebGroup                 I SRVE0169I: Loading Web Module: SimpleSessionWeb.
[9/16/23, 7:48:48:047 UTC] 0000006b com.ibm.ws.webcontainer                                      I SRVE0250I: Web Module SimpleSessionWeb has been bound to default_host.
[9/16/23, 7:48:48:048 UTC] 0000006b com.ibm.ws.http.internal.VirtualHostImpl                     A CWWKT0016I: Web application available (default_host): http://simplesession-865dc67844-wnw4d:9080/SimpleSessionWeb/
[9/16/23, 7:48:48:117 UTC] 00000070 com.ibm.ws.session.WASSessionCore                            I SESN0176I: A new session context will be created for application key default_host/SimpleSessionWeb
[9/16/23, 7:48:48:122 UTC] 00000070 com.ibm.ws.util                                              I SESN0172I: The session manager is using the Java default SecureRandom implementation for session ID generation.
[9/16/23, 7:48:48:144 UTC] 00000058 org.infinispan.HOTROD                                        I ISPN004006: Server sent new topology view (id=-192856701, age=0) containing 1 addresses: [10.254.20.18/<unresolved>:11222]
[9/16/23, 7:48:48:151 UTC] 00000058 org.infinispan.HOTROD                                        I ISPN004014: New server added(10.254.20.18/<unresolved>:11222), adding to the pool.
[9/16/23, 7:48:48:153 UTC] 00000058 org.infinispan.HOTROD                                        I ISPN004016: Server not in cluster anymore(example-infinispan/<unresolved>:11222), removing from the pool.
[9/16/23, 7:48:48:234 UTC] 00000073 org.infinispan.HOTROD                                        I ISPN004006: Server sent new topology view (id=-192856701, age=0) containing 1 addresses: [10.254.20.18/<unresolved>:11222]
[9/16/23, 7:48:48:239 UTC] 00000073 org.infinispan.HOTROD                                        I ISPN004014: New server added(10.254.20.18/<unresolved>:11222), adding to the pool.
[9/16/23, 7:48:48:240 UTC] 00000073 org.infinispan.HOTROD                                        I ISPN004016: Server not in cluster anymore(example-infinispan/<unresolved>:11222), removing from the pool.
[9/16/23, 7:48:48:246 UTC] 00000071 com.ibm.ws.webcontainer.osgi.mbeans.PluginGenerator          I SRVE9103I: A configuration file for a web server plugin was automatically generated for this server at /opt/ibm/wlp/output/defaultServer/logs/state/plugin-cfg.xml.
[9/16/23, 7:48:48:291 UTC] 00000070 com.ibm.ws.cache.CacheServiceImpl                            I DYNA1056I: Dynamic Cache (object cache) initialized successfully.
[9/16/23, 7:48:48:330 UTC] 00000070 com.ibm.ws.app.manager.AppMessageHelper                      A CWWKZ0001I: Application SimpleSession started in 0.576 seconds.
[9/16/23, 7:48:48:351 UTC] 0000003a com.ibm.ws.tcpchannel.internal.TCPPort                       I CWWKO0219I: TCP Channel defaultHttpEndpoint has been started and is now listening for requests on host *  (IPv6) port 9080.
[9/16/23, 7:48:48:352 UTC] 0000003a com.ibm.ws.tcpchannel.internal.TCPPort                       I CWWKO0219I: TCP Channel defaultHttpEndpoint-ssl has been started and is now listening for requests on host *  (IPv6) port 9443.
[9/16/23, 7:48:48:353 UTC] 0000003a com.ibm.ws.tcpchannel.internal.TCPPort                       I CWWKO0219I: TCP Channel wasJmsEndpoint546 has been started and is now listening for requests on host localhost  (IPv4: 127.0.0.1) port 7276.
[9/16/23, 7:48:48:353 UTC] 0000003a com.ibm.ws.tcpchannel.internal.TCPPort                       I CWWKO0219I: TCP Channel wasJmsEndpoint546-ssl has been started and is now listening for requests on host localhost  (IPv4: 127.0.0.1) port 7286.
[9/16/23, 7:48:48:360 UTC] 0000003a com.ibm.ws.kernel.feature.internal.FeatureManager            A CWWKF0012I: The server installed the following features: [appClientSupport-1.0, appSecurity-2.0, batch-1.0, beanValidation-1.1, cdi-1.2, concurrent-1.0, distributedMap-1.0, ejb-3.2, ejbHome-3.2, ejbLite-3.2, ejbPersistentTimer-3.2, ejbRemote-3.2, el-3.0, j2eeManagement-1.1, jacc-1.5, jaspic-1.1, javaMail-1.5, javaee-7.0, jaxb-2.2, jaxrs-2.0, jaxrsClient-2.0, jaxws-2.2, jca-1.7, jcaInboundSecurity-1.0, jdbc-4.1, jms-2.0, jndi-1.0, jpa-2.1, jpaContainer-2.1, jsf-2.2, json-1.0, jsonp-1.0, jsp-2.3, managedBeans-1.0, mdb-3.2, servlet-3.1, sessionCache-1.0, ssl-1.0, wasJmsClient-2.0, wasJmsSecurity-1.0, wasJmsServer-1.0, webProfile-7.0, websocket-1.1].
[9/16/23, 7:48:48:361 UTC] 0000003a com.ibm.ws.kernel.feature.internal.FeatureManager            I CWWKF0008I: Feature update completed in 2.807 seconds.
[9/16/23, 7:48:48:361 UTC] 0000003a com.ibm.ws.kernel.feature.internal.FeatureManager            A CWWKF0011I: The defaultServer server is ready to run a smarter planet. The defaultServer server started in 5.196 seconds.
[9/16/23, 7:48:57:528 UTC] 00000069 .ibm.ws.transport.iiop.server.security.CSIv2SubsystemFactory E NO_USER_REGISTRY 
                                                                                                               defaultOrb
                                                                                                               10
% 
```




As you see below, the session data was maintained even after the pod restart.   
```
% curl -c cookie.txt http://simplesession-e30532.apps.*.*.*.*.com/SimpleSessionWeb/SimpleServlet
% curl -b cookie.txt http://simplesession-e30532.apps.*.*.*.*.com/SimpleSessionWeb/SimpleServlet
% curl -b cookie.txt http://simplesession-e30532.apps.*.*.*.*.com/SimpleSessionWeb/SimpleServlet
% oc exec -it simplesession-865dc67844-wnw4d  -- cat /logs/messages.log | grep SystemO                      
[9/16/23, 7:51:55:453 UTC] 00000071 SystemOut                                                    O first request. count: 0
[9/16/23, 7:52:04:806 UTC] 0000004a SystemOut                                                    O first request. count: 0
[9/16/23, 7:52:10:811 UTC] 00000048 SystemOut                                                    O t7hONC-IDgXbw0umbvmzXpG: count: 1
[9/16/23, 7:52:21:111 UTC] 0000003a SystemOut                                                    O t7hONC-IDgXbw0umbvmzXpG: count: 2
% oc get pod                                                                          
NAME                                                 READY   STATUS    RESTARTS   AGE
example-infinispan-0                                 1/1     Running   0          92m
example-infinispan-config-listener-79b9fd76f-qfq4m   1/1     Running   0          92m
simplesession-865dc67844-wnw4d                       1/1     Running   0          4m1s
% oc delete pod simplesession-865dc67844-wnw4d
pod "simplesession-865dc67844-wnw4d" deleted
% oc get pod                                                                                                
NAME                                                 READY   STATUS    RESTARTS   AGE
example-infinispan-0                                 1/1     Running   0          93m
example-infinispan-config-listener-79b9fd76f-qfq4m   1/1     Running   0          92m
simplesession-865dc67844-ghgjp                       1/1     Running   0          16s
% curl -b cookie.txt http://simplesession-e30532.apps.*.*.*.*.com/SimpleSessionWeb/SimpleServlet
% oc exec -it simplesession-865dc67844-ghgjp  -- cat /logs/messages.log | grep SystemO
[9/16/23, 7:53:13:370 UTC] 00000066 SystemOut                                                    O t7hONC-IDgXbw0umbvmzXpG: count: 3
% 
```
The cache is created at the DG side automatically during the startup of liberty if it doesn't exist in advance.   
<img width="1185" alt="image" src="https://github.com/e30532/RHDG/assets/22098113/e9dd2417-ea67-484a-8709-1e2d0f4a8f03">


Note: You may wonder if the remote liberty outside of OCP can interact with the DG on OCP. The remote liberty can connect to the DG through the exposed route like the standalone java, but once the DG library at the liberty side connects to the DG server, it retrieves the DG server topology information. With that information, liberty tries to connect to the infinispan cluster member. But the network path to the each cluster members is not exposed, the remote liberty can't establish the connection.   

