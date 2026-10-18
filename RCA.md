# Root Cause Analysis

Fill this in as you go. Bullet points are fine — be specific, not long.

---

## Defect 1: Proxy image fails to build
### Symptom / error output
- run-tests.sh failed while building the proxy image, exit code 100
- the last lines only said apt-get failed: "E: Sub-process /usr/bin/dpkg returned an error code (1)"

### How I diagnosed it
- - ran `./scripts/run-tests.sh` and scrolled up through the whole output
- a few lines above the error, dpkg was asking a question:
  "Configuration file '/etc/nginx/nginx.conf' , File on system created by you or by a script , (Y/I/N/O/D/Z) [default=N] ?"
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
- 127.0.0.1:8080 works from the host because backend publishes 8080 there, but inside the proxy container it's the proxy itselft

### Fix
- file changed: proxy/nginx.conf
- changed the upstream to `server backend:8080;`. "backend" is the compose service name, Docker's DNS resolves it to the backend container on the compose network, and 8080 is the port the backend listens on inside its container
- checked: the error now shows `upstream: "http://172.20.0.2:8080/api/info"`, so nginx goes to the backend container. it still got connection refused, but that's a different problem (defect 3)

---

## Defect 3: Tests still fail after fixing defect 2
### Symptom / error output
- after the routing fix, /api/info still failed with "HTTP Error 502: Bad Gateway"
- but now the proxy log showed the right address: `upstream: "http://172.20.0.2:8080/api/info"`, still "connect() failed (111: Connection refused)"
- so nginx reached the backend container, but nothing was listening there yet

### How I diagnosed it
- compared the timing in the run-tests.sh output: the test-runner sent its request while the backend was still printing "loaded config key:" lines, before "Backend listening on port 8080"
- read backend/app.py: load_config() sleeps CONFIG_FETCH_SECONDS (default 2) for each of the 4 config keys, and only after that it creates the HTTP server. so the backend needs about 8 seconds before it can answer anything
- docker-compose.yml used plain `depends_on: - backend`, which only waits for the container to be started, not for the app inside to be ready

### Root cause
- race condition between startup order and readiness
- compose started the proxy and test-runner as soon as the backend container started, and the tests (timeout=3, single attempt) ran during the ~8 seconds when the backend wasn't listening yet
- on a fast or slow machine the result could change, which is why it has to be fixed by waiting for the backend's state and not by timing

### Fix, and why I chose it over the alternatives
- file changed: docker-compose.yml
- added a healthcheck to the backend that checks its own /health with python (the image has no curl)
- the proxy now waits for the backend to be healthy (`condition: service_healthy`) instead of just started
- checked: tests start only after the backend is listening. ran it 5 times, all passed

Other options:
- `CONFIG_FETCH_SECONDS=0`: passes, but changes the third-party service and hides the problem
- `sleep` in the script: waits for time, not for the backend. can still fail on a slow machine

---

## Defect 4: CI pipeline fails

### Symptom / error output
- GitHub Actions failed on the "Execute test suite" step:
  `./scripts/run-tests.sh: Permission denied`, `Process completed with exit code 126`
- 126 means the file was found but couldn't be executed

### How I diagnosed it
- saw the same thing locally right after cloning: `./scripts/run-tests.sh` gave Permission denied
- checked what git stores for the file: `git ls-files -s scripts/run-tests.sh` showed `100644`, no execute bit

### Root cause
- the script was committed without the execute bit
- CI does a fresh checkout, so it gets the file exactly as git stores it, and `./` needs execute permission
- it can work on one dev machine after a local chmod, but that's not saved anywhere, so the next clone breaks again

### Fix
- file changed: scripts/run-tests.sh (only its mode, not its content)
- `git update-index --chmod=+x scripts/run-tests.sh`, so git now stores it as `100755`
- checked: `git ls-files -s` shows `100755`, and CI is green

Other options:
- adding `chmod +x` as a step in ci.yml: CI would pass, but anyone who clones still gets Permission denied
- changing ci.yml to `bash scripts/run-tests.sh`: also works, but the README says to run `./scripts/run-tests.sh`, so the file itself should be executable

---

## Bonus questions

Optional. Answer only the ones you got to — number them, skip the rest.

>1. Smaller proxy image
- before: `ubuntu:24.04` + nginx, curl, htop, vim, git: 443MB 
- after: `nginx:1.27-alpine` with only our nginx.conf copied in: 73.6MB
- removed curl, htop, vim and git: none of them are needed for nginx to run.
- pinned the version (1.27) instead of latest, so the build doesn't change on its own
- measured with `docker images`, tests still pass

---

## Anything you would do differently with more time
- add a healthcheck to the proxy and make test-runner wait for it.
- add a `.gitattributes` with `*.sh text eol=lf`, so a clone on Windows can't break the script with CRLF line endings
- CI runs twice for a branch with an open PR (push + pull_request). I'd limit push to main