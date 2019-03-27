# Workshop gateway

## Pre requisite

- Docker 
- APIs to use as upstream / micro-services

## Run the gateway

You can check the `docker-compose.yml` for additional details

```bash
docker-compose up
```

## Test the gateway 

```
# Main api
curl http://127.0.0.1:8000

# Admin api
curl http://127.0.0.1:8001
```

## Enable ....
