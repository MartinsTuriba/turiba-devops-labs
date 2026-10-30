# Lab 6 · Test what you ship, ship a release

GitHub user: MartinsTuriba · Reference pipeline used (Part 0 fallback): no

<!-- Keep every heading as it is: the grader finds the sections by name. Replace every blank (four underscores), and <you> in the first line and in section 3. -->

## 1 · Smoke test

- The names of my smoke-test steps in the `image` job: "Build for the smoke test (amd64, kept on the runner)" and "Smoke test the image".
- What my smoke test asks, and what makes it fail: it starts the image and asks `GET /` on port 5000, with ten retries one second apart. No answer after that, and the step prints `docker logs smoke` and fails.
- 🔮 Part 1, before my first run: will my code pass the smoke test? I predicted yes, because `GET /` answers without a database. What happened: it passed; the first `curl` got "Connection reset by peer" and the retry a second later got `{"name":"course-api","version":"0.1.0"}`.
- 🔮 Part 2, before the checks finished: which checks will go red after `break.sh`? I predicted only `image`, because the other three run the source, and the commit changes what goes into the image.

## 2 · What broke and how I found it

- The check and the step that went red: `image`, in the step "Smoke test the image".
- The log line that showed the cause (copied from the log):

```text
Error: Cannot find module './banner'
```

- The cause, in my own words: the commit made `server.js` load `./banner`, but it also added `banner.js` to `api/.dockerignore`, so the file never reached the build context and the image didn't have it. Node crashed on start, so nothing answered on port 5000.
- Why `lint`, `unit` and `integration` stayed green: they run the source on the runner, where `banner.js` exists. Nothing before the smoke test ever started the image.
- My fix, and the commit that made it: I first deleted the whole `.dockerignore` (eb36ced), which turned the check green but would have put `.env` and `node_modules` into the image. ffeaea2 restored the file without the two banner lines.

## 3 · Release

- Release page: https://github.com/MartinsTuriba/turiba-devops-labs/releases/tag/v0.2.0
- Image: `ghcr.io/martinsturiba/turiba-devops-labs/api:0.2.0`, platforms (from `docker buildx imagetools inspect`): linux/amd64, linux/arm64
- One thing in Part 3 that didn't work the first time, and how I fixed it: nothing failed; the notes listed every PR, because this fork had no earlier tag.
