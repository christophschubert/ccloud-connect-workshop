# ccloud-connect-workshop
Setting up a self-managed Kafka Connect cluster with Confluent Cloud.

# Set up Confluent Cloud

Create a cluster and a schema registry.

You might also want to start the DataGen source connector.

## Add credentials

Edit the `.env` file to add your Kafka bootstrap server and API key/secret as well as
your schema registry url and API key/secret.

* Be sure not to confuse Kafka and schema registry credentials.
* Use a Kafka API key with 'global access'.

## Bring up connect cluster
You should be able to start the Connect cluster:

```bash
docker-compose up -d
```

It can take a couple of minutes for the connect cluster to fully start.
You could repeatedly use

```bash
curl http://localhost:8083
```
until you see a non-404 response.

Have a look at the Confluent Cloud UI: you should now see three Connect metadata topics (all starting with `self-managed-connect-`).

Use

```bash
curl http://localhost:8083/connectors
```
to see a list of deployed connectors. It should be empty.

Use

```bash
curl http://localhost:8083/connector-plugins
```
to see a list of currently available connectors. Recent version of the docker images contain only a minimal selection of connectors.

## Install the JDBC sink connector

We will now install the JDBC sink connector from [Confluent Hub](https://www.confluent.io/hub/).

```bash
docker compose exec kafka-connect confluent-hub install --no-prompt confluentinc/kafka-connect-jdbc:10.3.3
```
In a production environment, you would most likely build a custom docker image with the connector build in.

We can double check that the connector has been downloaded:
```bash
ls connect-plugins
```
but it has not been loaded by Connect yet (check `curl http://localhost:8083/connector-plugins`).
For this, we need to restart Kafka connect:

```bash
docker compose restart kafka-connect
```
After about 1 minute we should be able to see our new added plugins:
```bash
curl http://localhost:8083/connector-plugins
```


## Deploy the first connector
```bash
curl -X POST -d @jdbc-sink-simple.json  http://localhost:8083/connectors -H 'Content-Type: application/json'
```

Double-check it has been deployed:
```bash
curl http://localhost:8083/connectors
```
and check its status:
```bash
curl http://localhost:8083/connectors/simple-jdbc-sink/status
```

Let's look whether we are actually writing data:

```bash
docker compose exec postgres psql postgres postgres
```
```sql
\dt
select * from "users-avro";
```

At this point we might realize that having a dash in table-name isn't really convenient.
Moreover, we suddenly  remember a requirement to mask the user_id before writing out the data.

Both things can be fixed by using SMTs.

## Deploy connector with SMTs

```bash
curl -X POST -d @jdbc-sink-with-smt.json  http://localhost:8083/connectors -H 'Content-Type: application/json'
```

After deploying, we should double-check the status and then check in the database that we have indeed changed the tablename.
