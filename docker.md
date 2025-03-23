# docker

## environment variables
- add environment variables to run command with `-e` argument.

`docker run -e APP_COLOR=green simple-webapp-color`

## dockerfile entrypoint and cmd
- entrypoint is used when you want to pass additional arguments
```dockerfile
ENTRYPOINT ["sleep"]
```
- cmd is used to pass argument when you have entrypoint
```dockerfile
ENTRYPOINT ["sleep"]
CMD ["5"]
```
- can override entrypoint with `--entrypoint` argument when running image

## network
- `none` container cannot reach outside and cannot be reached complete isolation
- `host` container attached to host network no network isolation
- `bridge` docker creates internal network for attaching containers
  - on host interface created with `docker0` name
- list networks with `docker network ls`
- docker creates network namespace for each container