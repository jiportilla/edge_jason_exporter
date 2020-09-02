Description of problem:

**Bind for 0.0.0.0:9080 failed: port is already allocated**

How reproducible:
After Edge Registration.

Actual Results:
The container fails to start.

Expected Results:
Container starts.

Additional info:

Tail of /var/log/upstart/docker.log (after restarting Docker):

Resolution:
Another docker container was still running in the background from a different project.

This can be fixed by running:

```
docker stop $(docker ps -a -q)
docker rm $(docker ps -a -q)