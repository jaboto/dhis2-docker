# Jaime DHIS2 Docker
Repo on how to get DHIS2 working using Docker. This has been implemented already but here we describe the steps for training purposes.

## Getting ready

### Get Docker

The idea is to translate the official documentation: https://docs.dhis2.org/2.34/en/dhis2_system_administration_guide/installation.html into a set of containers (db: for postgres, dhis: for dhis2). This requires a set of steps described below.

### Images

dhis (URL)
mdillon/postgis (URL)

### Databases

Download the database from https://www.dhis2.org/downloads and put them in a specific directory for each version. This helps later

```
 jaime@ixalan  ~/uio/dbs  tree 
.
├── 2.33
│   └── 2.33_sl.sql.gz
└── 2.34
    └── 2.34_sl.sql.gz

```


## Running DHIS with DOCKER - Pure Docker Approach

### Network

Create a network so they can communicate with name (no need for hardcoded IP in the config files)

docker network create --driver bridge dhis-net

### Configuration files

#### Postgresql
The following postgresql file is the minimu file from the defaults plus the recommended in the manual, however it is not required as the only things that must be indicated to operate properly is the max_locks_per_transaction which ca be passed while spinning up the container. The file is listed here just in case a heavier configuration would be needed.
```
listen_addresses = '*'
max_connections = 200			
shared_buffers = 128MB			
work_mem = 20MB				
maintenance_work_mem = 512MB	
dynamic_shared_memory_type = posix
synchronous_comit = off
wal_writer_delay = 10000ms
random_page_cost = 1.1
max_locks_per_transaction = 96
log_timezone = 'UTC'
timezone = 'UTC'
lc_messages = 'en_US.utf8'		
lc_monetary = 'en_US.utf8'	
lc_numeric = 'en_US.utf8'
lc_time = 'en_US.utf8'	
default_text_search_config = 'pg_catalog.english'
```

#### dhis.conf
The configuration file is located under conf/dhis.conf and contains

```
connection.driver_class = org.postgresql.Driver
connection.url = jdbc:postgresql://db:5432/dhis2 # This is the other containers name and port
connection.username = dhis
connection.password = dhis
#server.https = on
#server.bsae.url = https://server.com/
```

### Run containers (name, image, network, other)

Run the database container:

* Use mdillon/postgis as it is a pure Postgresql with postigs (use alpine tag as its smaller and enough)
* Use POSTGRES variables user, password and db
* Expose the port 5432 (as defined in dhis.conf)
* Attach it to the previoulsy created network dhis-net
* Other flags: --rm to be deleted on stop, --name db to ease afterwards uses of the container, -d to dettach

```
docker run --rm --name db -e POSTGRES_PASSWORD=dhis -e POSTGRES_USER=dhis -e POSTGRES_DB=dhis2 -p 5432:5432 --network dhis-net mdillon/postgis:10-alpine postgres -c max_locks_per_transaction=96

```

Run the DHIS2 container:

* Use dhis2/core:(release tag)
* Expose the port 8080
* Copy the configuration file to the specifc folder defined by DHIS_HOME
* Attach it to the previoulsy created network dhis-net
* Other flags: --rm to be deleted on stop, --name db to ease afterwards uses of the container, -d to dettach

```
docker run --name dhis --rm -d -p 8080:8080 -v $(pwd)/dhis.conf:/DHIS2_home/dhis.conf --network dhis-net dhis2/core:2.34.0

```

... and in the same way we could run the database container with a downloaded Database, because of this it is a good practice to have each downloaded database in a different folder so they can be mapped as volume

```
docker run --rm --name db -e POSTGRES_PASSWORD=dhis -e POSTGRES_USER=dhis -e POSTGRES_DB=dhis2 -p 5432:5432 --network dhis-net -v /home/jaime/uio/dbs/2.34/:/docker-entrypoint-initdb.d/ mdillon/postgis:10-alpine -c 'max_locks_per_transaction=96'

```

## Running DHIS with Docker - Docker-compose approach

When using a Dockerfile we can specify some dependencies but this might not be enough. In this case *core* depends on *db* but this dependency is not real (as it only focuses on knowing if the container is up, but maybe the service is not up). This affects us because if we are loading a database at the beginning postgresql will be ready much earlier that the import of the database is done.

In docker-compse Version 2.1 the depends_on healthy checks works but it was removed in version 3 because of not taking the distributed approach. Some tools have been built to overcome this issue  like *dockerize* or *wait-for-it.sh* which, for our luck, *wait-for-it.sh* has been included in the core Docker image and can be used via the environment variable *WAIT-FOR-DB-CONTAINER*

```
version: '2.1'
services:
    dhis:
        image: dhis2/core:2.32.5
        ports:
            - "8080:8080"
        volumes:
            - /home/jaime/uio/docker/jaime-dhis2-docker/conf/dhis.conf:/DHIS2_home/dhis.conf
        environment:
            - WAIT_FOR_DB_CONTAINER=db:5432 -t 0
        depends_on:
            - db
    db:
        image: mdillon/postgis:10-alpine
        ports:
            - "5432:5432"
        volumes:
            - /home/jaime/uio/dbs/2.32/:/docker-entrypoint-initdb.d/
        command: postgres -c max_locks_per_transaction=96
        environment:
            - POSTGRES_PASSWORD=dhis
            - POSTGRES_USER=dhis
            - POSTGRES_DB=dhis2

```

Here we are starting to run into limitations:

* As you can see we are using the container's alias in the dhis.conf and therefor we can only run one set of dockers at a time. `connection.url = jdbc:postgresql://db:5432/dhis2` . We need to modify this per set of containers
* We are exposing the ports 8080 and 5432; we need to change them manually. Or have an entry container as reverse proxy (not so right form the application container perspective... :( )
* ... we could copy this docker-file multiple times
* ... or have multiple configuration files (one per version and included that in `dhis: volumes: definition`
* ... or have a single postgresql container with several databases (this is not so right from the application container perspective)
* ... or, or, or? Too much!



```
jaime@ixalan  ~/uio/docker/jaime-dhis2-docker  tree composers 
composers
├── 2.31
│   ├── conf
│   │   ├── dhis.conf
│   │   └── postgresql.conf
│   └── docker-compose.yml
├── 2.32
│   ├── conf
│   │   ├── dhis.conf
│   │   └── postgresql.conf
│   └── docker-compose.yml
└── 2.33
    ├── conf
    │   ├── dhis.conf
    │   └── postgresql.conf
    └── docker-compose.yml

6 directories, 9 files

```


... but then we can only run one set of containers per version, etc. If we are fine with this limitations, we probably are, we are done. If not, we can use the tool explained in the following section.

## Running DIHS2 with Docker - d2


## Monitoring
docker stats

