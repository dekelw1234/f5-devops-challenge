# Generative AI Disclosure

Using AI tools for this challenge is allowed. Be honest here — an accurate
"I used AI heavily" is worth more to us than an inaccurate "I did not".

**Did you use AI tools?** (yes / no): yes

**Which ones?** (e.g. ChatGPT, Claude, Copilot, Cursor): claude

> If you answered **no**, write "N/A" in sections 1 and 2 and skip to
> section 3. That is a perfectly good answer.

---

## 1. Key prompts

List the main prompts you used. Paraphrasing is fine.

1. Before the assignment: "teach me Docker, nginx, Docker Compose and GitHub Actions from scratch, step by step". -  We built a similar practice project (nginx + Flask + compose + CI), and I asked it to plant bugs in it for me to find and fix: https://github.com/dekelw1234/nginx-flask-compose
2. "what does this error / exit code mean?" for every error I hit (exit codes 100, 126, 137, 502, connection refused)
3. questions about parts of the assignment files I didn't understand, like the `upstream` block in nginx.conf and what `depends_on` actually waits for. I asked for hints, not solutions
4. "how do I write a healthcheck in a compose file?", which is what fixed defect 3. my first attempt had mistakes (no http://, wrong endpoint), so I asked for a working version and chose the minimal one out of the two it suggested
5. "how are file permissions saved in git, how do I see them and how do I change them?", for defect 4
6. "which packages in the proxy Dockerfile do I actually need?", for the image size bonus
7. after solving the tasks on my own, I redid them on a clean repo with Claude, reviewing each fix and writing the RCA. I asked it to rewrite the RCA in simpler, shorter language and then edited it myself

---

## 2. Where the AI got it wrong

Describe **one concrete case** where the AI's answer was wrong, outdated, or
sub-optimal. Cover: what it suggested, how you noticed it was wrong, and what
you did instead. 3–5 sentences.

> For defect 3, Claude first suggested healthchecks on both the backend and the proxy, with the test-runner waiting for the proxy.
> I asked if that was needed, since the real problem was only the backend's ~8 second startup, and nginx is ready almost immediately.
> I went with the minimal fix: a healthcheck on the backend and `condition: service_healthy` on the proxy. 
> It passed 5 runs in a row.

---

## 3. Anything in your submission you could not fully explain?

We would rather know. Write "nothing" if that is the case.

> the python one-liner in the backend healthcheck: I know Docker only looks at the exit code, and that urlopen fails if the backend doesn't answer, but I'm less sure about the details of how python exits on an error
> in nginx.conf, the `upstream` block and the `proxy_set_header` I didn't use them beyond this setup
> why nginx:alpine doesn't need apt-get: I know nginx comes already installed in the official image, but I'm less familiar with Alpine and its package manager
> exit codes: I don't know all of them by heart. For the ones in this assignment (100, 126, 137) I looked up what each one means while debugging
