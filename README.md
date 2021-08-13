# iPC Platform deployment

### Introduction

This repository provides guidelines for the deployment of the iPC Platform, and it will be updated according to the platform development. 

In the broad sense, the iPC Platform is formed by a Data Management/Catalogue system integrated with an analysis platform. Besides, additional components have been designed for establishing proper communication channels and interfaces between these systems, while providing mechanisms for a secure data access within the platform. Components are organized as microservices, allowing them to have an independent development process and easing their orchestrated deployment. [More info about the platform](https://ipc-project.eu/wp-content/uploads/2021/01/iPC-D2.2-PU-M22.pdf).

At a technical level, the iPC Data Catalogue is based on [Arranger](https://www.overture.bio/), and also, tailored made components have been built for adding new functionalities on top of it. The [Catalogue-Outbox-API](https://github.com/inab/iPC-Catalogue-Outbox-API-v2), provides a way to externalize user selections on the [Data Catalogue portal](https://github.com/inab/iPC_Data_Portal), which is accessible to external applications/data consumers (e.g: Data Analysis platform - [Virtual Research Environment](https://vre.ipc-project.bsc.es)). Additionally, the [Permissions-API](https://github.com/inab/Permissions-API) has been put in place for storing and dispatching user's dataset permissions. Importantly, the iPC platform has implemented an OpenIDConnect flow through all the platform components, using Keycloak as AuthN/Z server. Finally, submodules are tied to a CI/CD workflow (CircleCI) for automatic testing and delivery, and therefore, easing collaboration and deployment (Pull-based: Docker registry + Watchtower).

### Dependencies:
- Docker and docker-compose

### Services:
- Elasticsearch
- Kibana
- Arranger (server and ui)
- Data Catalogue portal
- Catalogue Outbox API
- Permissions API
- MongoDB
- Watchtower

### How to deploy?

- Clone the [project's repository](https://github.com/acavalls/arranger/tree/deploy):

    ```
    git clone https://github.com/acavalls/arranger.git
    ```

- Checkout to the project's branch:

    ```
    git checkout deploy
    ```

- Initialise git submodules (Data Catalogue portal, Catalogue Outbox API, Permissions API) 

    Execute the following commands in the root folder:

    ```
    git submodule init
    git submodule update
    ```

    As a result, the different git repositories will be cloned as dependencies of the main project.

- Configure Keycloak:

    You will need to configure Keycloak clients for the different submodules services (Data Catalogue Portal, Catalogue Outbox API, Permissions API). 

    - Data Catalogue Portal:

        * Public client.

    - Permissions API : 

        * Confidential client.
        * Enable service accounts.
        * Create two roles (admin/user) at the client level.
        * Create two roles (admin-realm/user-realm) at the realm level, and create composite roles for your client (which means assigning admin -> admin-realm + user -> user-realm).

        Once configured, the Data Catalogue Portal (+others) will be able to consume Permissions API via OAuth2 (JWT) based on roles.

    - Catalogue Outbox API: 

        * Same steps as Permissions API.

- Configure the environment:

    Before initializing the stack, there are four environment files that must be configured. 

    - Data Catalogue portal
    - Catalogue Outbox API
    - Permissions API
    - Root folder

    There is an .env.example file on each of the submodules root folder, that should be renamed:

    ```
    mv .env.example .env
    ```

- Configure Apache/nginx web server:

    Here is an example on how to configure Apache for handling requests to the different services: 

>    ...
>
>    <VirtualHost *:443>
>
>	    ServerName your-domain
>
>	    LogLevel info
>            ErrorLog  ${APACHE_LOG_DIR}/your-domain_error.log
>            CustomLog ${APACHE_LOG_DIR}/your-domain_access.log combined
>
>	    SSLEngine On
>            SSLProxyEngine On
>	         SSLCACertificatePath    /etc/apache2/ssl/project/
>            SSLCertificateFile      /etc/apache2/ssl/project/any-name.crt
>            SSLCertificateKeyFile   /etc/apache2/ssl/project/any-name.decrypt.key
>            SSLCertificateChainFile /etc/apache2/ssl/project/any-name.crt
>
>        ProxyRequests Off
>        AllowEncodedSlashes On
>	
>        # Data Catalogue service.
>        <Location />
>            ProxyPass http://host-ip:5002/ connectiontimeout=1800 timeout=1800
>            ProxyPassReverse http://host-ip:5002/
>        </Location>
>
>        # Catalogue Outbox API service.
>        <Location /catalogue_outbox/api>
>            ProxyPass http://host-ip:8085 connectiontimeout=1800 timeout=1800
>            ProxyPassReverse http://host-ip:8085
>        </Location>
>
>        # Permissions API service.
>        <Location /permissions/api>
>            ProxyPass http://host-ip:8082 connectiontimeout=1800 timeout=1800
>            ProxyPassReverse http://host-ip:8082
>        </Location>
>
>        # Elasticsearch.
>	     <Location /es_host>
>	        ProxyPass http://host-ip:9200 connectiontimeout=1800 timeout=1800
>	        ProxyPassReverse http://host-ip:9200
>        </Location>
>
>        # Kibana
>	     <Location /kibana>
>	        ProxyPass http://host-ip:5601 connectiontimeout=1800 timeout=1800
>	        ProxyPassReverse http://host-ip:5601
>        </Location>
>
>        # Arranger server.
>	    <Location /arranger_api>
>	        ProxyPass http://host-ip:5050 connectiontimeout=1800 timeout=1800
>	        ProxyPassReverse http://host-ip:5050
>        </Location>
>
>    </VirtualHost>


- Launch the stack (main project root folder):


    * Development:

	    ```
	    docker-compose up -d
	    ```

    * Production:

	    ```
	    docker-compose -f docker-compose.production.yml up -d
	    ```

 

