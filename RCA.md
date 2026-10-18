# Root Cause Analysis

Fill this in as you go. Bullet points are fine — be specific, not long.

---

## Defect 1: Proxy image fails to build

### Symptom / error output
- run-tests.sh failed while building the proxy image, exit code 100
- the last lines only said apt-get failed: "E: Sub-process /usr/bin/dpkg returned an error code (1)"

### How I diagnosed it
- saved the full output with `./scripts/run-tests.sh 2>&1 | tee task1.log` so I could read all of it
- a few lines above the error, dpkg was asking a question:
  "Configuration file '/etc/nginx/nginx.conf' ... File on system created by you or by a script ... (Y/I/N/O/D/Z) [default=N] ?"
- then looked at proxy/Dockerfile and saw the COPY of nginx.conf happens before nginx is installed

### Root cause
- the nginx package comes with its own /etc/nginx/nginx.conf
- our COPY had already put a file there, so dpkg saw a conflict and asked what to keep
- during docker build there's no one to answer, so dpkg failed and apt-get returned 100
- -y didn't help because it only answers apt's "continue?" question, not dpkg's config prompt
- so the problem wasn't a missing package, it was the order of the two lines

### Fix
- file changed: proxy/Dockerfile
- moved `COPY nginx.conf /etc/nginx/nginx.conf` to after the `RUN apt-get install` line. nginx installs its default config first, then our COPY overwrites it, and COPY doesn't ask anything
- also better for caching: changing nginx.conf now doesn't trigger reinstalling all the packages
- checked: the proxy image builds and the script gets to the tests

---

## Defect 2: Proxy cannot reach the backend


### Symptom / error output
- after fixing the build, /nginx-health passed but /api/info failed
- test-runner: "TEST RUNNER CRASHED: HTTP Error 502: Bad Gateway"

### How I diagnosed it
- looked at the proxy logs in the run-tests.sh output:
  "connect() failed (111: Connection refused) while connecting to upstream ... upstream: "http://127.0.0.1:8080/api/info""
- so nginx itself was fine (the health check it answers on its own passed), but it was sending requests to 127.0.0.1:8080
- checked proxy/nginx.conf: the upstream was `server 127.0.0.1:8080;`

### Root cause
- inside a container, 127.0.0.1 is that container itself
- the backend runs in its own container, so the proxy was trying to reach port 8080 on itself, where nothing listens
- since backend publishes 8080 to the host. but from inside the proxy container that's a different localhost

### Fix
- file changed: proxy/nginx.conf
- changed the upstream to `server backend:8080;`. "backend" is the compose service name, Docker's DNS resolves it to the backend container on the compose network, and 8080 is the port the backend listens on inside its container
- checked: the error now shows `upstream: "http://172.20.0.2:8080/api/info"`, so nginx goes to the backend container. it still got connection refused, but that's a different problem (defect 3)

---

## Defect 3: Tests still fail after fixing defect 2

### Symptom / error output

### How I diagnosed it

### Root cause

### Fix, and why I chose it over the alternatives

---

## Defect 4: CI pipeline fails

### Symptom / error output

### How I diagnosed it

### Root cause

### Fix

---

## Bonus questions

Optional. Answer only the ones you got to — number them, skip the rest.

>

---

## Anything you would do differently with more time
