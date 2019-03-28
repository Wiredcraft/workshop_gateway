# API Gateway with Kong - workshop

## Purpose

The purpose of this workshop is to get a hands-on approach to gateway API - in particular kong.
The examples below can easily be extended to support more complex routing, flow, ACL, etc.

## Pre requisite

- Docker
- APIs to use as upstream / micro-services; in this workshop we "cheat" our way through fake api (refer to the `./api/Dockerfile` for a better sense of it.)

## Run the gateway

You can check the `docker-compose.yml` for additional details

```bash
docker-compose up
```

## Test the gateway

```
# Main api
curl http://127.0.0.1:8000

{"message":"no Route matched with those values"}
```

```
# Admin api
curl http://127.0.0.1:8001
```

## Let's play !

### Glossary

- what is route
- service
- upstream
- target
- consumer
- plugins

More info: https://docs.konghq.com/1.0.x/admin-api/

### basic setup

**Feature**: routing

- 2 services pointing to 2 real API
- 2 route to route the traffic to that API

```
# Create services

curl -i -H "Content-Type: application/json" -X POST \
    http://127.0.0.1:8001/services \
    -d '{"name":"api1","url":"http://api1:8000"}'

curl -i -H "Content-Type: application/json" -X POST \
    http://127.0.0.1:8001/services \
    -d '{"name":"api2","url":"http://api2:8000"}'
```

```
# Create routes

curl -i -H "Content-Type: application/json" -X POST \
    http://127.0.0.1:8001/services/api1/routes \
    -d '{"hosts":["example.com"], "paths": ["/api1"]}'

curl -i -H "Content-Type: application/json" -X POST \
    http://127.0.0.1:8001/services/api2/routes \
    -d '{"hosts":["example.com"], "paths": ["/api2"]}'
```

Test:
- any other URL is rejected
- any hit of the route works

```
# Succeed
curl -i -H 'Host: example.com' http://127.0.0.1:8000/api1
curl -i -H 'Host: example.com' http://127.0.0.1:8000/api2

# Fails
curl -i -H 'Host: example.com' http://127.0.0.1:8000/api3
```

Advanced: routing can be done by:
- method
- host
- path 
- ...

### Load balancer

**Feature**: Load balancing

- 1 upstream
- 2 targets
- 1 service
- 1 route

```
# Create upstream

curl -i -H "Content-Type: application/json" -X POST \
    http://127.0.0.1:8001/upstreams \
    -d '{"name":"upstream_1"}'
```

```
# Add targets to upstream

curl -i -H "Content-Type: application/json" -X POST \
    http://127.0.0.1:8001/upstreams/upstream_1/targets \
    -d '{"target":"api1:8000"}'
curl -i -H "Content-Type: application/json" -X POST \
    http://127.0.0.1:8001/upstreams/upstream_1/targets \
    -d '{"target":"api2:8000"}'
```

```
# Create service using the upstream

curl -i -H "Content-Type: application/json" -X POST \
    http://127.0.0.1:8001/services \
    -d '{"name":"upstream","url":"http://upstream_1"}'
```

```
# Create route to route to the service
curl -i -H "Content-Type: application/json" -X POST \
    http://127.0.0.1:8001/services/upstream/routes \
    -d '{"hosts":["example.com"], "paths": ["/load-balance"]}'

```

Test:
- alternate response from 1 / 2

```
curl -i -H 'Host: example.com' http://127.0.0.1:8000/load-balance
```

### User and access control

**Feature**: access / key

- 2 consumers
- 2 api keys
- 2 groups
- acl plugin installed on each api to allow 1 group

```
# Create consumers

curl -i -H "Content-Type: application/json" -X POST \
    http://127.0.0.1:8001/consumers \
    -d '{"username": "user1"}'

curl -i -H "Content-Type: application/json" -X POST \
    http://127.0.0.1:8001/consumers \
    -d '{"username": "user2"}'
```

```
# Create keys (used through as auth-headers)

curl -i -H "Content-Type: application/json" -X POST \
    http://127.0.0.1:8001/consumers/user1/key-auth \
    -d '{"key": "key-of-user-1"}'

curl -i -H "Content-Type: application/json" -X POST \
    http://127.0.0.1:8001/consumers/user2/key-auth \
    -d '{"key": "key-of-user-2"}'
```

```
# Associate users with groups

# Associate to 1 group only
curl -i -H "Content-Type: application/json" -X POST \
    http://127.0.0.1:8001/consumers/user1/acls \
    -d '{"group": "group1"}'

# Associate to 2 groups
curl -i -H "Content-Type: application/json" -X POST \
    http://127.0.0.1:8001/consumers/user2/acls \
    -d '{"group": "group1"}'
curl -i -H "Content-Type: application/json" -X POST \
    http://127.0.0.1:8001/consumers/user2/acls \
    -d '{"group": "group2"}'
```

```
# Apply Auth plugin to service

curl -i -H "Content-Type: application/json" -X POST \
    http://127.0.0.1:8001/services/api1/plugins \
    -d '{"name": "key-auth", "config": {"key_names": ["api-key"]}}'

curl -i -H "Content-Type: application/json" -X POST \
    http://127.0.0.1:8001/services/api2/plugins \
    -d '{"name": "key-auth", "config": {"key_names": ["api-key"]}}'
```

```
# Apply ACL to service

curl -i -H "Content-Type: application/json" -X POST \
    http://127.0.0.1:8001/services/api1/plugins \
    -d '{"name": "acl", "config": {"whitelist": ["group1"]}}'

curl -i -H "Content-Type: application/json" -X POST \
    http://127.0.0.1:8001/services/api2/plugins \
    -d '{"name": "acl", "config": {"whitelist": ["group2"]}}'

```

Test:
- `/api1` available to user1 & user2 -- unavailable to othere
- `/api2` available to user 2 only 
- `/load-balance` still available to all

```
curl -i -H 'Host: example.com' -H 'api-key: key-of-user-1' http://127.0.0.1:8000/api1
curl -i -H 'Host: example.com' -H 'api-key: key-of-user-2' http://127.0.0.1:8000/api1

curl -i -H 'Host: example.com' -H 'api-key: key-of-user-1' http://127.0.0.1:8000/api2
curl -i -H 'Host: example.com' -H 'api-key: key-of-user-2' http://127.0.0.1:8000/api2

curl -i -H 'Host: example.com' -H 'api-key: key-of-user-1' http://127.0.0.1:8000/load-balance
curl -i -H 'Host: example.com' -H 'api-key: key-of-user-2' http://127.0.0.1:8000/load-balance
```

Advanced:
- support of jwt
- support of basic auth
- support of sso
- support of ....

### Apply rate limiting protection

Feature: rate limiting

- 1 rate limiting on 1 API

```
curl -i -H "Content-Type: application/json" -X POST \
    http://127.0.0.1:8001/plugins \
    -d '{"name": "rate-limiting", "config": {"minute": 10, "hour": 50}}'
```

Test:
- user 1: get blocked
- user 2: still has his quota
- anonymous: get blocked; user 1/2 still have access

```
# Reach rate limit threshold

curl -i -H 'Host: example.com' -H 'api-key: key-of-user-1' http://127.0.0.1:8000/api1
curl -i -H 'Host: example.com' -H 'api-key: key-of-user-2' http://127.0.0.1:8000/api1
curl -i -H 'Host: example.com' -H 'api-key: key-of-user-1' http://127.0.0.1:8000/load-balance

ab -n 100 -H 'Host: example.com' http://127.0.0.1:8000/api1
```

Advanced:
- rate limit conrtrol from backend (response rate-limit)

### Logs

**Feature**: logs

- hit the APIs

```
# Add file-log plugin

curl -i -H "Content-Type: application/json" -X POST \
    http://127.0.0.1:8001/plugins \
    -d '{"name": "file-log", "config": {"path": "/logs/trace.log"}}'
```

Test:
- see the trace logs + extra headers from auth (x-consumer-id for authenticated users)

### Modify requests / responses

**Feature**: transformation

- remove request header (security - e.g. x-secret)
- remove response header (obfuscation)

```
# Add request-transformer plugin

curl -i -H "Content-Type: application/json" -X POST \
    http://127.0.0.1:8001/plugins \
    -d '{
        "name": "request-transformer", 
        "config": {
            "remove": {
                "headers": ["x-to-remove", "other-header"]
            }
        }
    }'

# Add response transformer

curl -i -H "Content-Type: application/json" -X POST \
    http://127.0.0.1:8001/plugins \
    -d '{
        "name": "response-transformer", 
        "config": {
            "remove": {
                "headers": ["x-ratelimit-limit-minute"]
            }
        }
    }'
```

Test:
- before: we can see the x-rate-limit
- after: invisible

Check the data/kong/trace.log - notice the logs only show the `keep-header` - notice also the response to the curl request is missing the `x-ratelimit-limit-minute` header.

```
curl -i -H 'Host: example.com' -H 'api-key: key-of-user-1' http://127.0.0.1:8000/api1 -H 'keep-header: test' -H 'x-to-remove: byebye'
```


Advanced:
- request/response-transformer can do much more than remove headers, it can fully rewrite headers, querystring, body, etc.

