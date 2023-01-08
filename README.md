# Elastic stack (ELK) on Docker & Connect MongoDB to Elasticsearch via Logstash
In this tutorial, we use the ``JDBC`` plugin to communicate
## Jdbc plugin
This plugin was created as a way to ingest data in any database with a JDBC interface into Logstash. You can periodically schedule ingestion using a cron syntax (see schedule setting) or run the query one time to load data into Logstash. Each row in the resultset becomes a single event. Columns in the resultset are converted into fields in the event. [Read more](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-jdbc.html)

## Download the latest version of the Jdbc Mongo plugin
:inbox_tray: [Download Plugin](https://dbschema.com/jdbc-driver/MongoDb.html)

## docker-compose.yml
```yml
version: '3.7'

services:

  # The 'setup' service runs a one-off script which initializes users inside
  # Elasticsearch — such as 'logstash_internal' and 'kibana_system' — with the
  # values of the passwords defined in the '.env' file.
  #
  # This task is only performed during the *initial* startup of the stack. On all
  # subsequent runs, the service simply returns immediately, without performing
  # any modification to existing users.
  setup:
    build:
      context: setup/
      args:
        ELASTIC_VERSION: ${ELASTIC_VERSION}
    init: true
    volumes:
      - ./setup/entrypoint.sh:/entrypoint.sh:ro,Z
      - ./setup/helpers.sh:/helpers.sh:ro,Z
      - ./setup/roles:/roles:ro,Z
      - setup:/state:Z
    environment:
      ELASTIC_PASSWORD: ${ELASTIC_PASSWORD:-}
      LOGSTASH_INTERNAL_PASSWORD: ${LOGSTASH_INTERNAL_PASSWORD:-}
      KIBANA_SYSTEM_PASSWORD: ${KIBANA_SYSTEM_PASSWORD:-}
      METRICBEAT_INTERNAL_PASSWORD: ${METRICBEAT_INTERNAL_PASSWORD:-}
      FILEBEAT_INTERNAL_PASSWORD: ${FILEBEAT_INTERNAL_PASSWORD:-}
      HEARTBEAT_INTERNAL_PASSWORD: ${HEARTBEAT_INTERNAL_PASSWORD:-}
      MONITORING_INTERNAL_PASSWORD: ${MONITORING_INTERNAL_PASSWORD:-}
      BEATS_SYSTEM_PASSWORD: ${BEATS_SYSTEM_PASSWORD:-}
    networks:
      elk:
        ipv4_address: 192.168.55.20
    depends_on:
      - elasticsearch
  


  elasticsearch:
    build:
      context: elasticsearch/
      args:
        ELASTIC_VERSION: ${ELASTIC_VERSION}
    volumes:
      - ./elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro,Z
      - elasticsearch:/usr/share/elasticsearch/data:Z
    ports:
      - 9200:9200
      - 9300:9300
    environment:
      node.name: elasticsearch
      ES_JAVA_OPTS: -Xms512m -Xmx512m
      # Bootstrap password.
      # Used to initialize the keystore during the initial startup of
      # Elasticsearch. Ignored on subsequent runs.
      ELASTIC_PASSWORD: ${ELASTIC_PASSWORD:-}
      # Use single node discovery in order to disable production mode and avoid bootstrap checks.
      # see: https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html
      discovery.type: single-node
    networks:
      elk:
        ipv4_address: 192.168.55.21

  logstash:
    build:
      context: logstash/
      args:
        ELASTIC_VERSION: ${ELASTIC_VERSION}
    volumes:
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml:ro,Z
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro,Z
      - ./logstash/jars:/usr/share/logstash/logstash-core/lib/jars
    ports:
      - 5044:5044
      - 50000:50000/tcp
      - 50000:50000/udp
      - 9600:9600
    environment:
      LS_JAVA_OPTS: -Xms256m -Xmx256m
      LOGSTASH_INTERNAL_PASSWORD: ${LOGSTASH_INTERNAL_PASSWORD:-}
    networks:
      elk:
        ipv4_address: 192.168.55.22
    depends_on:
      - elasticsearch

  kibana:
    build:
      context: kibana/
      args:
        ELASTIC_VERSION: ${ELASTIC_VERSION}
    volumes:
      - ./kibana/config/kibana.yml:/usr/share/kibana/config/kibana.yml:ro,Z
      - ./monogojs/mongo-init.js:/docker-entrypoint-initdb.d/mongo-init.js:ro
    ports:
      - 5601:5601
    environment:
      KIBANA_SYSTEM_PASSWORD: ${KIBANA_SYSTEM_PASSWORD:-}
    networks:
      elk:
        ipv4_address: 192.168.55.23
    depends_on:
      - elasticsearch


  mongo:
    image: mongo
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: 123456
    networks:
      elk:
        ipv4_address: 192.168.55.24
    ports:
      - "27017:27017"
      - "28017:28017"
    volumes:
      - ./mongojs/mongo-init.js:/docker-entrypoint-initdb.d/mongo-init.js:ro

  # Database Manager
  mongo-express:
    image: mongo-express
    ports:
      - 8099:8081
    depends_on:
      - mongo
    environment:
      ME_CONFIG_BASICAUTH_USERNAME: express
      ME_CONFIG_BASICAUTH_PASSWORD: 123456
      ME_CONFIG_MONGODB_PORT: 27017
      ME_CONFIG_MONGODB_ADMINUSERNAME: root
      ME_CONFIG_MONGODB_ADMINPASSWORD: 123456
    links:
      - mongo

networks:
  elk:
    driver: bridge
    ipam:
     config:
       - subnet: 192.168.55.0/24
         gateway: 192.168.55.1


volumes:
  setup:
  elasticsearch:

```

## logstash.conf
```javascript
input {
	jdbc {
		jdbc_driver_library => "/usr/share/logstash/logstash-core/lib/jars/mongojdbc4.8.jar"
		jdbc_driver_class => "Java::com.wisecoders.dbschema.mongodb.JdbcDriver"
		jdbc_connection_string => "jdbc:mongodb://root:123456@192.168.55.24:27017/sample_db?authSource=admin"
		jdbc_user => "root"
    jdbc_password => "123456"
		schedule => "* * * * * *"
		statement => "db.sample_collection.find({},{'_id': false})" 
	}
}
output {
	elasticsearch {
		hosts => ["http://192.168.55.21:9200"]
		user => "elastic"
		password => "changeme"
		data_stream => auto
		index => "mymongodb-%{+YYYY-MM-dd}"
		action => "index"
	}
}
```

## Run docker-comose
```
docker-compose up -d
```
## View index created in Kibana
1. open your web browser at [kibana](http://localhost:5601) :rocket:
2. go on sidebar and select ``Dev Tools`` in Management section
3. Run this query => `` GET mymongodb-{{write your Date}}/_search`` :tada:

## Down docker-compose
```
docker-compose down
```
