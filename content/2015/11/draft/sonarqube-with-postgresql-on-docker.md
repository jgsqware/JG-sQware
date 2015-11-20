+++
Categories = []
Description = ""
Tags = []
date = "2015-11-20T11:52:33+01:00"
title = "SonarQube with PostgreSQL on Docker"

+++

# Run SonarQube on PostgreSQL

Use this `docker-compose.yml`

```yaml
sonarqube:
  image: sonarqube
  ports:
   - "9000:9000"
   - "5432:5432"
  links:
    - db:db
  environment:
   - SONARQUBE_JDBC_URL=jdbc:postgresql://db:5432/sonar

db:
  image: postgres
  environment:
   - POSTGRES_USER=sonar
   - POSTGRES_PASSWORD=sonar
```

then

```bash
> docker-compose up -d
```

Then your SonarQube instance is available on `<DOCKER-MACHINE-IP>:9000`.

# Analyse your project

```bash
> mvn sonar:sonar \
  -Dsonar.host.url=http://<DOCKER-MACHINE-IP>:9000 \
  -Dsonar.jdbc.url=jdbc:postgresql://<DOCKER-MACHINE-IP>/sonar
```

# Backup Sonar data

```bash
> docker run --rm --link sonar_db_1:db -v $(pwd):/data -e PGPASSWORD="sonar" postgres-backup -h db -U sonar -f /data/sonar_db_backup.sql
```

# Restore Sonar data

```bash
> docker run --rm --link sonar_db_1:db -v $(pwd):/data -e PGPASSWORD="sonar" postgres-restore -h db -U sonar -f /data/sonar_db_backup.sql
```
