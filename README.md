# Introduction

This project exposes a simple Secured REST endpoint where the service `greeting` is available, but properly secured, at this address `http://hostname:port/greeting` and returns a json Greeting message after authentication

```
{
    "content": "Hello, World!",
    "id": 1
}

```

The id of the message is incremented for each request. To customize the message, you can pass as parameter the name of the person that you want to send your greeting.

The project is spilt into two Apache Maven modules - `app` and `build`.
The `App` module exposes the REST Service using as technology Spring Boot bundled with the Apache Tomcat 8.0.36 artifacts while the `build` module contains the OpenShift objects
required to deploy the Red Hat SSO Server 7.0 along with the "app" module.

The goal of this project is to deploy the quickstart against an OpenShift environment (online, dedicated, ...).

# Build

In order to build and deploy this project, it is required to have an account on an OpenShift Online (OSO): https://console.dev-preview-stg.openshift.com/ instance
or you're welcome to setup your own OpenShift env; via minishift.

Once you have this, along with the [OpenShift CLI tool](https://docs.openshift.com/online/cli_reference/get_started_cli.html), you're ready to go.

Open a terminal, log on to the OpenShift Server `oc login https://<OPENSHIFT_ADDRESS> --token=MYTOLEN` when you use OpenShift Online or Dedicated.

Create a new project on OpenShift `oc new-project <some_project_name>` and next build the quickstart 

```
mvn clean install
```

# Launch / deploy

To deploy the whole secured app, first move to build/ dir, and then simply use the `Fabric8` Maven Plugin with the goals `deploy` and `start`:

```
cd build
mvn fabric8:deploy fabric8:start
```

Open OpenShift console in the browser to see the status of the app,
and the exact routes, to be used to access the app's greeting endpoint or to access the Red Hat SSO's admin console.

Note: until https://issues.jboss.org/browse/CLOUD-1166 is fixed,
we need to fix the redirect-uri in RH-SSO admin console, to point to our app's route.

To specify the Red Hat SSO URL to be used by the springBoot Application, it is required to change the SSO_URL env variable assigned to the DeploymentConfig object.
You can change this value using the following oc command where the https server to be defined corresponds to the location of the Red Hat SSO Server running 
in OpenShift.

```
oc env dc/secured-springboot-rest SSO_URL=https://secure-sso-sso.e8ca.engint.openshiftapps.com/auth
```

# Access the service

If the pod of the Secured Spring Boot Application is running like also the Red Hat SSO Server, you 
can use one of the bash scripts proposed within the root of the project to access the service.

Depending which tool you prefer to use (curl or httpie), use one of bash files available and pass as parameters
the address of the Red Hat Secured SSO Server and the Secured Spring Boot Application. 

```
./scripts/httpie/token_req.sh https://secure-sso-sso.e8ca.engint.openshiftapps.com http://springboot-rest-sso.e8ca.engint.openshiftapps.com
./scripts/curl/token_req.sh https://secure-sso-sso.e8ca.engint.openshiftapps.com http://springboot-rest-sso.e8ca.engint.openshiftapps.com
```

The URLs of the Red Hat SSO & Spring Boot Application are created according to this convention:

* Red Hat Secured SSO : <secured_sso_route>.<namespace>.<host_machine>
* Secured Spring Boot Application : <secured_springboot_route>.<namespace>.<host_machine>

You can find such routes using this oc client command `oc get routes` or the Openshift Console.

# Access the service using a user without admin role

To secure the Spring Boot REST endpoint, different properties must be defined within the `app/src/main/resources/application.properties` file which contains
such Keycloak parameters. 

```
keycloak.securityConstraints[0].securityCollections[0].name=admin stuff
keycloak.securityConstraints[0].securityCollections[0].authRoles[0]=admin
keycloak.securityConstraints[0].securityCollections[0].patterns[0]=/greeting
```

The patterns property defines as pattern, the `/greeting` endpoint which means that this endpoint is protected by Keycloak. Every other endpoint that is not explicitly listed is NOT secured by Keycloak and is publicly available.
The authRoles property defines which Keycloak roles are allowed to access the defined endpoints. Typically, the default `admin` user which is used as the `admin` role and will be able to access the service.

To verify that a user without the `admin` role can't access the service, you will create a new user using the following bash script

```
./scripts/curl/add_user.sh <SSO_HOST> <SpringBoot_HOST>
./scripts/httpie/add_user.sh <SSO_HOST> <SpringBoot_HOST>
```

Next, you can call again the greeting endpoint by issuing a HTTP request where the username is `bburke` and the password `password`. In response, you will be notified that yoou can't access to the service

```
./scripts/curl/token_user_req.sh <SSO_HOST> <SpringBoot_HOST>
./scripts/httpie/token_user_req.sh <SSO_HOST> <SpringBoot_HOST>
```

# Test

(atm, we exclude / ignore tests, as they hit RH-SSO which we don't have running as part of the tests)

To test locally the quickstart, install & start a Red Hat SSO server. Next, pass as parameter the URL to access the SSO Server 

```
mvn test -Dsso.url=http://localhost:8080
```