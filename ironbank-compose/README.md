#### Example Compose Configuration Using IronBank Images

This repository contains an example docker-compose configuration that works with the identified version(s) of Anchore Enterprise 
images from IronBank.


This compose file is configured for use with Anchore Enterprise 4.0.2

##### Prerequisites
This example assumes you have the following:
* Docker and docker-compose installed
* Anchore Enterprise license file
* Platform One Harbor account for retrieving IronBank hardened images
* Internet access (the feed service will attempt to retrieve vulnerability data)

It's important to understand that every effort is taken to remain parity between the Anchore Enterprise images
and the IronBank approved Anchore images. That said, one key difference is the presence of environment variables within
the created images. The IronBank images remove several default environment variables and are required.  To address this,
the missing ENV vars need to be specified which can be accomplished by added variables to the services within the 
`environment` section for each service in the compose file. For example, the following shows the values for the 
`rbac-manager` service.

```yaml
  rbac-manager:
       environment:
          - ANCHORE_AUTH_PRIVKEY=null
          - ANCHORE_AUTH_SECRET=null
          - ANCHORE_ADMIN_PASSWORD=foobar
          - ANCHORE_ENTERPRISE_FEEDS_GITHUB_DRIVER_TOKEN=null
          - ANCHORE_ENDPOINT_HOSTNAME=rbac-manager
          - ANCHORE_DB_HOST=anchore-db
          - ANCHORE_DB_PASSWORD=mysecretpassword
          - ANCHORE_AUTHZ_HANDLER=external
          - ANCHORE_EXTERNAL_AUTHZ_ENDPOINT=http://rbac-authorizer:8228
          - ANCHORE_ENABLE_METRICS=false
          - ANCHORE_LOG_LEVEL=INFO
```

As of this writting the following `ENV` variables are removed from the IronBank images and must be specified as 
described above.

```text
ANCHORE_ADMIN_PASSWORD                         # requires a value
ANCHORE_CLI_PASS                               # requires a value
ANCHORE_AUTH_PRIVKEY                           # can be null
ANCHORE_AUTH_SECRET                            # can be null
ANCHORE_ENTERPRISE_FEEDS_GITHUB_DRIVER_TOKEN   # can be null
```

##### Ironbank Images:

* registry1.dso.mil/ironbank/anchore/enterprise/enterprise:4.0.2

* registry1.dso.mil/ironbank/anchore/enterpriseui/enterpriseui:4.0.2

* registry1.dso.mil/ironbank/opensource/postgres/postgresql12:12.11

The Anchore Enterprise docker-compose quick start can be run with minimal modification for the purposes of getting
started quickly. To do that, you could simply clone this repository, ensure the license file is available and run:

Note: this requires being logged in to the registry1.dso.mil registry on the node where the compose command is run.

```
 $ docker-compose up -d
```
This should result in the following output:

```shell
Creating network "ironbank-compose_default" with the default driver
Creating volume "ironbank-compose_anchore-enterprise-4.0.2-db" with default driver
Creating volume "ironbank-compose_feeds-workspace-volume" with default driver
Creating volume "ironbank-compose_enterprise-feeds-db-volume" with default driver
Pulling catalog (registry1.dso.mil/ironbank/anchore/enterprise/enterprise:)...
latest: Pulling from ironbank/anchore/enterprise/enterprise
Digest: sha256:3aecc91ea3a422f3db1a693c543aa39db41e1045d16d713eb7ca563099a5f3a0
Status: Downloaded newer image for registry1.dso.mil/ironbank/anchore/enterprise/enterprise:latest
Creating ironbank-compose_enterprise-feeds-db_1 ... done
Creating ironbank-compose_anchore-db_1          ... done
Creating ironbank-compose_ui-redis_1            ... done
Creating ironbank-compose_feeds_1               ... done
Creating ironbank-compose_catalog_1             ... done
Creating ironbank-compose_rbac-authorizer_1     ... done
Creating ironbank-compose_rbac-manager_1        ... done
Creating ironbank-compose_api_1                 ... done
Creating ironbank-compose_notifications_1       ... done
Creating ironbank-compose_reports_1             ... done
Creating ironbank-compose_queue_1               ... done
Creating ironbank-compose_policy-engine_1       ... done
Creating ironbank-compose_analyzer_1            ... done
Creating ironbank-compose_ui_1                  ... done
```

You can check the status be doing the following:
```shell
 $ docker-compose ps
 
                 Name                               Command                  State                        Ports
---------------------------------------------------------------------------------------------------------------------------------
ironbank-compose_analyzer_1              /docker-entrypoint.sh anch ...   Up (healthy)   8228/tcp
ironbank-compose_anchore-db_1            docker-entrypoint.sh postgres    Up (healthy)   5432/tcp
ironbank-compose_api_1                   /docker-entrypoint.sh anch ...   Up (healthy)   0.0.0.0:8228->8228/tcp,:::8228->8228/tcp
ironbank-compose_catalog_1               /docker-entrypoint.sh anch ...   Up (healthy)   8228/tcp
ironbank-compose_enterprise-feeds-db_1   docker-entrypoint.sh postgres    Up (healthy)   5432/tcp
ironbank-compose_feeds_1                 /docker-entrypoint.sh anch ...   Up (healthy)   0.0.0.0:8448->8228/tcp,:::8448->8228/tcp
ironbank-compose_notifications_1         /docker-entrypoint.sh anch ...   Up (healthy)   0.0.0.0:8668->8228/tcp,:::8668->8228/tcp
ironbank-compose_policy-engine_1         /docker-entrypoint.sh anch ...   Up (healthy)   8228/tcp
ironbank-compose_queue_1                 /docker-entrypoint.sh anch ...   Up (healthy)   8228/tcp
ironbank-compose_rbac-authorizer_1       /docker-entrypoint.sh anch ...   Up (healthy)   8089/tcp, 8228/tcp
ironbank-compose_rbac-manager_1          /docker-entrypoint.sh anch ...   Up (healthy)   0.0.0.0:8229->8228/tcp,:::8229->8228/tcp
ironbank-compose_reports_1               /docker-entrypoint.sh anch ...   Up (healthy)   0.0.0.0:8558->8228/tcp,:::8558->8228/tcp
ironbank-compose_ui-redis_1              docker-entrypoint.sh redis ...   Up (healthy)   6379/tcp
ironbank-compose_ui_1                    /docker-entrypoint.sh node ...   Up (healthy)   0.0.0.0:3000->3000/tcp,:::3000->3000/tcp
```
And finally to check the status of the services you can run:

```shell
$ anchore-cli --u admin --p foobar --url http://localhost:8228/v1 system status
Service reports (anchore-quickstart, http://reports:8228): up
Service analyzer (anchore-quickstart, http://analyzer:8228): up
Service simplequeue (anchore-quickstart, http://queue:8228): up
Service notifications (anchore-quickstart, http://notifications:8228): up
Service rbac_authorizer (anchore-quickstart, http://rbac-authorizer:8228): up
Service policy_engine (anchore-quickstart, http://policy-engine:8228): up
Service apiext (anchore-quickstart, http://api:8228): up
Service rbac_manager (anchore-quickstart, http://rbac-manager:8228): up
Service catalog (anchore-quickstart, http://catalog:8228): up

Engine DB Version: 0.0.17
Engine Code Version: 4.0.0
```
You can also navigate to `http://localhost:3000` in your browser to see the Anchore Enterprise user interface and use the 
default admin user and password (set by the env var above) to log in.

### Configuring Feeds
This section shows how the deployment configuration can be customized based on your specific needs.  Each service configuration 
has a `volumes` section which mounts the license file and an optional config.yaml file.

```yaml
    volumes:
      - ./license.yaml:/license.yaml:ro
      #- ./config-enterprise.yaml:/config/config.yaml:z
```
If desired, we can create and provide a configuration file to customize the service behavior. In a Kubernetes deployment
this would be accomplished by tailoring the values.yaml file provided to the helm deployment. Here, we'll show how to 
customize the feeds by specifying we want to provided the `feed-config.yaml` provided in this repository to the feed 
service configuration in our docker-compose.

We do this by uncomment the line that reads `- ./feed-config.yaml:/config/config.yaml:z`. See below.

```yaml
  feeds:
    image: registry1.dso.mil/ironbank/anchore/enterprise/enterprise
    volumes:
      - feeds-workspace-volume:/workspace
      - ./license.yaml:/license.yaml:ro
      - ./feed-config.yaml:/config/config.yaml:z
```

If we look inside the `feed-config.yaml` we'll see a section named `drivers` which is where we adjust what vulnerability
feed drivers we want to turn on (or off). Below  we show a portion of this file and we can see where we've enabled `rhel` 
and `nvdv2`. Which means all other feeds will not be updated.

```yaml
drivers:
  # Configuration section for drivers gathering and normalizing feed data.
  amzn:
    enabled: false
  alpine:
    enabled: false
  centos:
    enabled: false
  debian:
    enabled: false
  ol:
    enabled: false
  ubuntu:
    enabled: false
  rhel:
    enabled: true
  nvddb:
    # disabled driver, a newer nvdv2 driver is available
    enabled: false
  npm:
    # npm driver is disabled out of the box
    enabled: "${ANCHORE_ENTERPRISE_PACKAGE_DRIVERS_ENABLED}"
  gem:
    # rubygems driver is disabled out of the box
    enabled: "${ANCHORE_ENTERPRISE_PACKAGE_DRIVERS_ENABLED}"
    # rubygems package data is a PostgreSQL dump. db_connect string defines the database endpoint to be used for loading the data
    db_connect: 'postgresql://${ANCHORE_DB_USER}:${ANCHORE_DB_PASSWORD}@${ANCHORE_DB_HOST}:${ANCHORE_DB_PORT}/${ANCHORE_RUBYGEMS_DB_NAME}'
  nvdv2:
    enabled: true
  vulndb:
    enabled: false
  msrc:
    enabled: false
```

Now, if you ran the started the compose stack as described earlier, you can now update your stack with this new configuration
by running.

```shell
$ docker-compose up --force-recreate -d
```
This will recreate the containers and inject the `feed-config.yaml` into the feeds service `ironbank-compose_feeds_1`.
We can verify this by performing the following which will list the contents of the config.yaml file inside the feeds container
which was created from the `feed-config.yaml` we specified.

```shell
$ docker exec -it ironbank-compose_feeds_1 cat /config/config.yaml 
```

At this point, it will take some time to download the vulnerability data (on the order of hours). After some time, you 
can run the following to retrieve a list of feeds being synced to your system.

```shell
  acli --u admin --p foobar --url http://localhost:8228/v1 system feeds list
```

#### Cleanup
When done, you can simple run the following to tear down the stack and clean up all containers and volumes.

```shell
$ docker-compose down --volumes
```
