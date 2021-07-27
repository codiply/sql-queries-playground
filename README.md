# SQL Queries Playground

## Docker containers

Run

```
docker-compose up -d
```

Depending on the size of the data, it might take long to populate the database. Follow the logs

```
docker-compose logs -f
```

or check the cpu usage to see when it is done

```
docker stats
```

Remove the containers

```
docker-compose down
```

Force rebuild

```
docker-compose build --no-cache
```

## Credentials

- Server: `localhost` or `db` (for adminer on `localhost:8080`)
- Database: `demo`
- User: `admin`
- Password: `password`

## Queries

See [here](queries.md)