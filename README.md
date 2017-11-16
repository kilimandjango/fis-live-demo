# fis-live-demo

**Requirements**
- CDK 3.x or OpenShift
- Git
- Mvn

**Setup project**

*Example:*
```
$ mvn org.apache.maven.plugins:maven-archetype-plugin:2.4:generate \
  -DarchetypeCatalog=https://maven.repository.redhat.com/ga/io/fabric8/archetypes/archetypes-catalog/2.2.195.redhat-000013/archetypes-catalog-2.2.195.redhat-000013-archetype-catalog.xml \
  -DarchetypeGroupId=org.jboss.fuse.fis.archetypes \
  -DarchetypeArtifactId=spring-boot-camel-xml-archetype \
  -DarchetypeVersion=2.2.195.redhat-000013
```
When processing the archetype setup
- _groupid_ to "org.example.fis"
- _artefactid_ to "fis-spring-boot"

Now switch to the created project

**Setup Git Repository**

```
$ cd fis-spring-boot
```
Create a new repository on your Git server.
Next initialize your repo and add your sources:
```
$ git init
$ git add *
```
Now you can commit your sources, sync your repository and push it to the server:
```
$ git commit -m "Initialize FIS Spring Boot project!"
$ git remote add origin https://github.com/kilimandjango/fis-demo.git
$ git push origin master
```
**Create project via OpenShift Web Console**

- Create a new project with the following parameters:
Name=fuse-demo
Display Name=fuse-demo

- Browse Catalog and choose Java Language

- Select Red Hat JBoss fuse-demo

- Select fis-java-openshift Version 2.0

- Choose a Name and put in your git repository

- Create the application

- Check build logs

- When push is successful check the deployment logs

**Setup Nexus**

In your project (check it with $oc project) create a Nexus:
```
$ oc new-app sonatype/nexus
```
Then create a new route
```
$ oc expose svc/nexus
```
Check the new route
```
$ oc get routes
```
Check the deployment and the web console:
```
https://<appname>-<namespace>.<subdomain>/nexus
```
Port is not needed because the router is proxying you to the correct endpoint!

**Setup Storage for Nexus**
Check available persistent volumes:
```
$ oc get pv
```
When available persistent volumes are unbound attach a volume to the Nexus deploymentconfig:
```
$ oc volumes dc/nexus --add \
	--name 'nexus-volume-1' \
	--type 'pvc' \
	--mount-path '/sonatype-work/' \
	--claim-name 'nexus-pv' \
	--claim-size '1G' \
	--overwrite
```
Now your Nexus is persistent!

**Configure the necessary proxy repositories**
Login to Nexus with _admin/admin123_

Add the following repositories to Nexus as proxy repositories:
fuse-source	https://repo.fusesource.com/nexus/content/groups/public/
jboss-ea-repository	https://maven.repository.redhat.com/earlyaccess/all/
jboss-ga-repository	https://maven.repository.redhat.com/ga/

Add the repositories to the group _public repositories_

Do not forget to refresh!

**Configure the project to use Nexus mirror**
In your git repository go to the folder configuration and add the following lines to the settings.xml above the profiles:
```
<mirrors>
  <mirror>
    <id>internal-repository</id>
    <name>Maven Repository Manager running on repo.mycompany.com</name>
    <url>${MVN_MIRROR}</url>
    <mirrorOf>*</mirrorOf>
  </mirror>
</mirrors>

```
Commit your changes. The settings.xml is automatically taken because of the implementation of the fis-java-s2i image! You don't have to care of configuring the mvn stuff.

In the OpenShift web console go to the build config of your Fuse application. Attach the following under sourceStrategy:
```
sourceStrategy:
       env:
     - name: BUILD_LOGLEVEL
       value: '5'
     - name: ARTIFACT_DIR
     - name: MAVEN_ARGS
       value: package -DMVN_MIRROR=http://nexus-fuse-demo.openshift.viada.de/nexus/content/groups/public/ -DskipTests -Dfabric8.skip -e -B
     - name: MAVEN_ARGS_APPEND
   from:
```
**Test Nexus connection**
Now rebuild your project and check the logs if the connection to Nexus mirror is working.

If it is working compare the build times!

ATTENTION: The first build is taking a quite long time, approximately about 15 minutes. The following builds should be a lot faster!

**Setup Jenkins**
Now configure Jenkins. In the OpenShift Web console add a Jenkins-ephemeral to your application. Since it is a xPaas Image it is already preconfigured!

Create a new pipeline element and configure the pipeline:

```
node{
    stage('build') {
      print 'build'
      openshiftBuild apiURL: '', authToken: "", bldCfg: "s2i-spring-boot-camel-xml", buildName: '', checkForTriggeredDeployments: 'true', commitID: '', env:
                            [[name: 'BUILD_URL', value: "${BUILD_URL}"]], namespace: '', showBuildLogs: 'true', verbose: 'false'
    }          

    stage('staging') {
      print 'stage'
      openshiftDeploy apiURL: '', authToken: '', depCfg: "s2i-spring-boot-camel-xml", namespace: '', verbose: 'false', waitTime: '300', waitUnit: 'sec'    
    }
}
```
Start the pipeline.

**Check also Java Console/Hawtio**
In your project go to the deployment and click on Java Console.
