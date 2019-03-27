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

```
# Service
## create service
curl -i -H "Content-Type: application/json" -X POST http://127.0.0.1:8001/services -d '{"name":"api1","url":"http://api1:8000"}'

curl -i -H "Content-Type: application/json" -X POST http://127.0.0.1:8001/services -d '{"name":"api2","url":"http://api2:8000"}'

# Route
## add route for service
curl -i -H "Content-Type: application/json" -X POST http://127.0.0.1:8001/services/api1/routes -d '{"hosts":["example.com"], "paths": ["/api1"]}'

curl -i -H "Content-Type: application/json" -X POST http://127.0.0.1:8001/services/api2/routes -d '{"hosts":["example.com"], "paths": ["/api2"]}'

## test
curl -i -H 'Host: example.com' http://127.0.0.1:8000/api1
curl -i -H 'Host: example.com' http://127.0.0.1:8000/api2
curl -i -H 'Host: example.com' http://127.0.0.1:8000/api3

ab -n 100 -H 'Host: example.com' http://127.0.0.1:8000/api1

# Plugin
## add rate-limiting plugin for service: api1
curl -i -H "Content-Type: application/json" -X POST http://127.0.0.1:8001/services/api1/plugins -d '{"name": "rate-limiting", "config": {"minute": 10}}'

ab -n 11 -H 'Host: example.com' http://127.0.0.1:8000/api1
curl -i -H 'Host: example.com' http://127.0.0.1:8000/api1

## enable key-auth plugin for service: api2
curl -i -H "Content-Type: application/json" -X POST http://127.0.0.1:8001/services/api2/plugins -d '{"name": "key-auth", "config": {"key_names": ["api-key"]}}'

## test
curl -i -H 'Host: example.com' http://127.0.0.1:8000/api2

# Consumer
## add consumer
curl -i -H "Content-Type: application/json" -X POST http://127.0.0.1:8001/consumers -d '{"username": "api2-consumer"}'

## add api-key for recently created consumer
curl -i -H "Content-Type: application/json" -X POST http://127.0.0.1:8001/consumers/api2-consumer/key-auth -d '{"key": "KSEhU2VzRNoqK7D9PbJSJZ4o3JqmlMO2"}'

## test
curl -i -H 'Host: example.com' -H 'api-key: KSEhU2VzRNoqK7D9PbJSJZ4o3JqmlMO2' http://127.0.0.1:8000/api2
```