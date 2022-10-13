### Docker image

```dockerfile
FROM nginx/unit:1.28.0-php8.1
```

Note:
NGINX Unit provides a docker image for php, you can use it as it is, or you can use it as a base
image.

---

### Configuration in Docker

```dockerfile
FROM nginx/unit:1.28.0-php8.1

COPY ./config/*.json /docker-entrypoint.d/
```

Note:
NGINX Unit also provides an easy way to initialize configuration in docker. Just put your
configuration and a file, and then copy it to /docker-entrypoint.d/, and the entrypoint will upload
that file to the Unit's endpoint.
