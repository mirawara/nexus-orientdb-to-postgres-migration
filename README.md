# Nexus Migration from Orient DB to Postgres

This guide outlines the process of migrating a Nexus Docker instance from OrientDB (deprecated since version 3.71) to PostgreSQL, with an intermediate step through H2. Follow the instructions step by step to ensure proper execution of the migration.

I decided to create this guide because the official documentation is incomplete and not always reliable. I've consolidated information from Stack Overflow, Reddit, GitHub issues, and the official documentation to provide a more comprehensive solution.

This guide is also suitable for migrating from H2 to Postgres, just skip the first part.


## 1. Preparation

1. If you haven't done so, update the Nexus3 Docker image to version 3.70.3 (the latest available for OrientDB).

## 2. Backup and Setup

1. Backup the `nexus.properties` file:
    ```bash
    docker cp nexus_container_name:/opt/sonatype/sonatype-work/nexus3/etc/nexus.properties .
    ```

2. Stop the container:
    ```bash
    docker-compose down
    ```

3. Backup the Nexus database folder

4. Modify the `docker-compose.yml` file:
    Add the mapping for the `nexus.properties` file:
    ```yaml
        volumes:
          - ./nexus.properties:/opt/sonatype/sonatype-work/nexus3/etc/nexus.properties
    ```

5. Start the container again:
    ```bash
    docker-compose up -d
    ```
## 3. Migration to H2

Why not migrate directly to Postgres?
Because before version 3.77, you can only do it with the Pro version. :)

1. Access the container:
    ```bash
    docker exec -ti nexus_container_name bash
    ```

2. From the Nexus web interface, in the Tasks section, perform the backup and store the last 4 `.bak` files in a folder.

3. Enter the backup folder and download the Nexus database migrator:
    ```bash
    curl -s -L -O https://download.sonatype.com/nexus/nxrm3-migrator/nexus-db-migrator-3.70.1-03.jar
    ```

4. Run the H2 database migration:
    ```bash
    java -Xmx2G -Xms2G -XX:+UseG1GC -XX:MaxDirectMemorySize=28672M -jar nexus-db-migrator-3.70.1-03.jar --migration_type=h2
    ```

5. Stop Nexus:
    ```bash
    /opt/sonatype/nexus/bin/nexus stop
    ```

6. If the `nexus.mv.db` file exists, copy it to the `/nexus-data/db` folder:
    ```bash
    cp nexus.mv.db /nexus-data/db
    ```

7. Exit the container

8. Stop the container again:
    ```bash
    docker-compose down
    ```

9. Add the following configuration in the `nexus.properties` file:
    ```properties
    nexus.datastore.enabled=true
    ```

10. Start the containers:
    ```bash
    docker-compose up -d
    ```

11. Check the logs to verify that the database was correctly migrated:
    ```bash
    docker logs nexus_container_name | grep H2
    ```

The database is now perfectly migrated to H2.

## 4. Upgrade to Version 3.77.1

Now, you could update to the latest version and it would work, but then you wouldn't be able to migrate to Postgres (I have no idea why they don't mention it in the docs). So, you need to use version 3.77, the version from which it's possible to migrate to Postgres without the Pro version.

1. Update the Nexus version to `3.77.1` and remove the mapping for the `nexus.properties` file.

2. Start the container:
    ```bash
    docker-compose up -d
    ```

If you don't have the default encryption key error, skip the following steps.

3. Create a `secret.json` file with the following structure:
    ```json
    {
      "active": "key",
      "keys": [
        {
          "id": "key",
          "key": "very secure and long password string"
        }
      ]
    }
    ```

4. Add the following environment variables in the `docker-compose.yml` file:
    ```yaml
    environment:
      - NEXUS_SECRETS_KEY_FILE=/opt/sonatype/secret.json
    volumes:
      - ./secret.json:/opt/sonatype/secret.json
    ```

5. Restart the container:
    ```bash
    docker-compose down && docker-compose up -d
    ```

6. Re-encrypt the secret key using the REST API (you have to change only password and hostname):
    ```bash
    curl -u admin:<password> -X 'PUT' \
      'http(s)://your.repo.com/service/rest/v1/secrets/encryption/re-encrypt' \
      -H 'accept: application/json' \
      -H 'Content-Type: application/json' \
      -H 'NX-ANTI-CSRF-TOKEN: token' \
      -H 'X-Nexus-UI: true' \
      -d '{
      "secretKeyId": "key"
    }'
    ```


## 3. Migration to Postgres

1. Add PostgreSQL to the `docker-compose.yml` file:
    ```yaml
      postgres:
          image: postgres:17.4
          hostname: db
          restart: on-failure
          volumes:
            - ./postgresql_data:/var/lib/postgresql/data
          environment:
            POSTGRES_DB: nexus
            POSTGRES_USER: nexus
            POSTGRES_PASSWORD: nexus
            PGDATA: /var/lib/postgresql/data/pgdata
            PGOPTIONS: -c max_connections=200
          healthcheck:
            test: ["CMD-SHELL", "sh -c 'pg_isready -U nexus -d nexus'"]
            interval: 30s
            timeout: 10s
            retries: 5

    ```
    Change the password of course... 

    `PGOPTIONS: -c max_connections=200` It's not required, but it's recommended.

2. Add this to nexus service in the `docker-compose.yml` file:
    ```yaml
        depends_on:
          postgres:
            condition: service_healthy
            restart: true
    ```

3. Start the containers:
    ```bash
    docker-compose up -d
    ```

3. Clean the backup directory

4. From the Nexus web interface, in the Tasks section, perform the backup and store the .zip file in a folder

5. Access the container:
    ```bash
    docker exec -ti nexus_container_name bash
    ```
6. Cd into the zip folder

7. Stop Nexus:
    ```bash
    /opt/sonatype/nexus/bin/nexus stop
    ```

8. Copy the Nexus database:
    ```bash
    cp /nexus-data/db/nexus.mv.db .
    ```

9. Download the Nexus database migrator:
    ```bash
    curl -s -L -O https://download.sonatype.com/nexus/nxrm3-migrator/nexus-db-migrator-3.77.1-01.jar
    ```

10. Run the migration from H2 to PostgreSQL:
    ```bash
    java -Xmx16G -Xms16G -XX:+UseG1GC -XX:MaxDirectMemorySize=28672M -jar nexus-db-migrator-*.jar --migration_type=h2_to_postgres --db_url="jdbc:postgresql://db:5432/nexus?user=nexus&password=nexus&currentSchema=public"
    ```
11. Exit the container

12. Access the postgres container:
    ```bash
    docker exec -ti postgres_container_name bash
    ```

13. Access PostgreSQL:
    ```bash
    psql -U nexus
    ```

14. Run the `VACUUM` command to optimize the database:
    ```sql
    VACUUM(FULL,ANALYZE,VERBOSE);
    ```

15. Check the `max_connections` configuration:
    ```sql
    SHOW max_connections;
    ```

    If it's not set to 200, set `max_connections` to 200 and delete the PGOPTIONS env variable from `docker-compose.yml` :
    ```sql
    ALTER SYSTEM SET max_connections = 200;
    ```

16. Exit the container

17. Stop the containers:
    ```bash
    docker-compose down
    ```

18. Modify the `docker-compose.yml` file by removing the mappings for `nexus-data` and `backup` folder (if any), and adding the following environment variables:
    ```yaml
    environment:
      - NEXUS_DATASTORE_NEXUS_JDBCURL: jdbc:postgresql://db:5432/nexus?currentSchema=public
      - NEXUS_DATASTORE_NEXUS_USERNAME: nexus
      - NEXUS_DATASTORE_NEXUS_PASSWORD: nexus
      - NEXUS_DATASTORE_NEXUS_ADVANCED: maximumPoolSize=190
    ```
    190 is arbitrary, the important thing is that it's not 200. If you've left the default value in Postgres (100), set a number lower than one hundred. Otherwise, in Postgres, you'll get "FATAL: sorry, too many clients already."


3. Start the containers:
    ```bash
    docker-compose up -d
    ```

    You may need to run `docker-compose down` and `docker-compose up -d` again.


## 5. Upgrade to Version 3.78.2 (or above)

# Troubleshooting

* If you encounter errors in the Java command, try removing the options `-Xmx2G -Xms2G -XX:+UseG1GC -XX:MaxDirectMemorySize=28672M`.

# Buy me a coffee
It took me a lot of time, study, swearing, and more to create this guide. I want to donate the result of my efforts to the community, but if you feel like thanking me, buy me a coffee! : [paypal.me/mirawara](https://paypal.me/mirawara)

