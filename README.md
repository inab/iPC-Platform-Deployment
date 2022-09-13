# iPC Platform deployment

### Summary:

This repository provides guidelines for the deployment of the iPC Platform, and it will be updated according to the platform development. 

In the broad sense, the iPC Platform is formed by a Data Management/Catalogue system integrated with an analysis platform. Besides, additional components have been designed for establishing proper communication channels and interfaces between these systems, while providing mechanisms for managing users data requests and datasets permissions within the platform. More info: [D2.2](https://ipc-project.eu/wp-content/uploads/2021/01/iPC-D2.2-PU-M22.pdf), [D2.3](https://docs.google.com/document/d/1EyA68KLUjE8ywhGEOSF-aGZD0njQK8ev/edit?usp=sharing&ouid=101486945169185784257&rtpof=true&sd=true), [D2.4](https://docs.google.com/document/d/1OhCeTi5wT6t29JjwbIvfRx4WPGGm-NZucvfIKJf1d7E/edit?usp=sharing).

### Components:
The iPC Data Catalogue is based on [Arranger](https://www.overture.bio/), and also, tailored made components have been built for adding new functionalities on top of it. The [Catalogue-Outbox-API](https://github.com/inab/iPC-Catalogue-Outbox-API-v2), provides a way to externalize user selections on the [Data Catalogue portal](https://github.com/inab/iPC_Data_Portal), which is accessible to external applications/data consumers (e.g: Data Analysis platform - [Virtual Research Environment](https://vre.ipc-project.bsc.es)). The [Permissions-API](https://github.com/inab/Permissions-API) has been put in place for storing and dispatching user's dataset permissions, which are assigned by the proper Data Access Committes through the [DAC-Portal](https://github.com/inab/DAC-Portal.git), where data policies can also be established. The Data Access Committees can be easily created in the [DAC Management Portal]("https://github.com/inab/DAC-Management-Portal.git"), where a group of users can be assigned specific resources to be controlled. The DAC Management Portal is integrated with [Nextcloud]("https://nextcloud.com/") a data storage system on which data owners can deposit their data right before the creation of a Data Access Committee. The [DAC-Notify](https://github.com/inab/DAC-Notify.git) service provides a notification system between applications and end-users triggered by events, which is based on [RabbitMQ](https://www.rabbitmq.com/) integrated with an SMTP service. [Keycloak](https://www.keycloak.org/) instance manages AuthN/Z, which not only provides a Single Sign-On login flow (OpenIDConnect) throughout the platform components, but also, a user-friendly roles management system.

### Operations:
Components are organized as microservices and independent repositories, which not only eases their orchestrated deployment (docker-compose) but also components reusability (git submodules). The submodules are tied to a CI/CD workflow (CircleCI) for automated testing and delivery, and therefore, facilitating collaborative work across teams and their final deployment in production (Pull-based: Docker registry + Watchtower).

### Dependencies:
- Docker and docker-compose

### Services:
- Elasticsearch
- Kibana
- Arranger (server and ui)
- Data Catalogue portal
- Catalogue Outbox API
- Data Access Committee Portal (DAC-Portal)
- Data Access Committee Management Portal (DAC-Management-Portal)
- Data Access Committee Notify (DAC-Notify)
- Nextcloud
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

- Initialise git submodules (Data Catalogue portal, Catalogue Outbox API, Data Access Committee Portal, Permissions API) 

    Execute the following commands in the root folder:

    ```
    git submodule init
    git submodule update
    ```

    As a result, the different git repositories will be cloned as dependencies of the main project.


- Configure the environment:

    Before initializing the stack, there are five environment files that must be configured. 

    - Data Catalogue portal
    - Catalogue Outbox API
    - Data Access Committee Portal (DAC-Portal)
    - Permissions API
    - Root folder

    There are .env-example-X files on each of the submodules root folder showing the configuration needed for their deployment on different environments. In any case, those files should be renamed:

    ```
    mv .env.example-X .env
    ```

- Configure Keycloak:

    In development or production environments you will need to configure Keycloak clients for the different submodules services (Data Catalogue Portal, Catalogue Outbox API, Data Access Committee Portal, Permissions API). 

    - Data Catalogue Portal:

        * Public client.

    - Permissions API : 

        * Confidential client.
        * Enable service accounts.
        * Create four roles (dac-admin/dac-member/is-dac/user) at the client level.
        * Create three roles (dac-admin-realm/dac-member-realm/user-realm) at the realm level. 
        * Create composite roles for your client (dac-admin + is-dac -> dac-admin-realm, dac-member + is-dac -> dac-member-realm, user -> user-realm).

        Once configured, the Data Catalogue Portal (+others) will be able to consume Permissions API via OAuth2 (JWT) based on roles.

    - Data Access Committe Portal (DAC-Portal): 

        * Confidential client: Same steps as Permissions-API (DAC-Portal follows the same roles definition).
        * Public client: Same steps as Data Catalogue Portal.

    - Catalogue Outbox API: 

        * Confidential client: user role is needed - Composite: user -> user-realm.

- Configure reverse proxy: Apache/nginx web server

    Here is an example on how to configure Apache for handling requests to the different services. In this example, two conf files are provided (Data Catalogue, Data Access Committee Portal): 

    - Data Catalogue conf file:

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

    
    - Data Access Committee Portal conf file:

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
    >        # Data Access Committee Portal service.
    >        <Location />
    >            ProxyPass http://host-ip:3080/ connectiontimeout=1800 timeout=1800
    >            ProxyPassReverse http://host-ip:3080/
    >        </Location>
    >
    >        # Data Access Committee portal API service.
    >        <Location /api>
    >            ProxyPass http://host-ip:9090 connectiontimeout=1800 timeout=1800
    >            ProxyPassReverse http://host-ip:9090
    >        </Location>
    >
    >    </VirtualHost>


- Launch the stack (main project root folder):

    * Local:

        The docker-compose.local.yml creates a private subnetwork (172.21.0.0/24) that assigns static IPs for the different services (ipam). No additional configuration is needed and the stack is bootstrapped with test data (ipc-test-data submodule).

        ```
        docker-compose -f docker-compose.local.yml up -d
        ```

    * Cloud (specific to the iPC project):

        Keycloak runs separately of the rest of the stack, and it must be configured following the guidelines provided in section: "Configure Keycloak". In the case of running the stack behind a reverse proxy apply the configuration provided in section: "Configure reverse proxy".

        * Development:

            In development mode, docker images are built from Dockerfiles, and services source code is mounted as docker volumes for faster development process. 

	        ```
	        docker-compose -f docker-compose.development.yml up -d
	        ```

        * Production:
            
            In production mode, docker images are retrieved from DockerHub, and they are automatically updated - Pull-based deployment strategy (Watchtower). 

	        ```
	        docker-compose -f docker-compose.production.yml up -d
	        ```

 

