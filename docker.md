# docker

# environment variables
- add environment variables to run command with `-e` argument.
```bash
docker run -e APP_COLOR=green simple-webapp-color
```

# dockerfile entrypoint and cmd
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