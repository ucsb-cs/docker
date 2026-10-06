# docker

Github Actions Scripts to generate Docker Images under the `ucsbcs` Docker Hub account.

**Contents**

- [PrairieLearn datascience images](#prairielearn-datascience-images)
- [Using the images in a PrairieLearn question](#using-the-images-in-a-prairielearn-question)
- [Where Python runs in PrairieLearn (and why there are three installs)](#where-python-runs-in-prairielearn-and-why-there-are-three-installs)
- [Symptoms: what it looks like when `datascience` is missing somewhere](#symptoms-what-it-looks-like-when-datascience-is-missing-somewhere)
- [Why the `datascience` version is pinned](#why-the-datascience-version-is-pinned)
- [How to update `datascience` everywhere](#how-to-update-datascience-everywhere)
- [How the images get built](#how-the-images-get-built)
- [Crucial PrairieLearn step: syncing images](#crucial-prairielearn-step-syncing-images)

# PrairieLearn datascience images

For CMPSC 5A and CMPSC 5B, the Python module `datascience` is crucial to the course
material, and it is not installed by default in PrairieLearn.

In order to use `datascience` in external graders and workspace questions,
we need to build custom docker images that:

* start with the images created by the PrairieLearn team
* add the `datascience` library, at a pinned version

In the folder `images`, you will find a subdirectory for each image. Under each of
those folders is a `Dockerfile` that does exactly this.

| Image | Base image | Role in PrairieLearn |
|---|---|---|
| `ucsbcs/grader-python-datascience` | `prairielearn/grader-python` | **External grader**: runs and tests student code submitted via `pl-file-editor` / `pl-file-upload` |
| `ucsbcs/workspace-jupyterlab-python-datascience` | `prairielearn/workspace-jupyterlab-python` | **Workspace**: the JupyterLab environment students work in for workspace questions |

Each image is published with two tags: `latest`, and the pinned `datascience`
version (e.g. `0.18.1`). See [Why the `datascience` version is pinned](#why-the-datascience-version-is-pinned).

# Using the images in a PrairieLearn question

For an **externally graded** question (student writes code, the autograder runs tests
against it), the question's `info.json` should contain:

```json
"gradingMethod": "External",
"externalGradingOptions": {
  "enabled": true,
  "image": "ucsbcs/grader-python-datascience:latest",
  "entrypoint": "/python_autograder/run.sh",
  "timeout": 20
}
```

For a **workspace** question (student works in JupyterLab in the browser):

```json
"workspaceOptions": {
  "image": "ucsbcs/workspace-jupyterlab-python-datascience:latest",
  "port": 8080,
  "home": "/home/jovyan",
  ...
}
```

Do not mix these up. The workspace image has **no autograder** in it
(there is no `/python_autograder/run.sh`), and the grader image has **no JupyterLab**.

# Where Python runs in PrairieLearn (and why there are three installs)

A single PrairieLearn question can execute Python in up to **three different
containers**, and `datascience` has to be installed in each one separately.
This is the single most common source of confusion, so here is the map.

```
                 ┌──────────────────────────────────────────────────────────┐
                 │ 1. prairielearn/prairielearn (the PL server itself)      │
  question       │    runs: questions/*/server.py, custom elements           │
  render/grade ─▶│    Python libs: PL's pyproject.toml                       │
                 │      + anything in  <course>/serverFilesCourse/           │
                 └──────────────────────────────────────────────────────────┘
                                         │
             (externally graded question) │ student submits code
                                         ▼
                 ┌──────────────────────────────────────────────────────────┐
                 │ 2. externalGradingOptions.image  (a grader container)    │
                 │    e.g. ucsbcs/grader-python-datascience                  │
                 │    runs: /python_autograder/run.sh -> tests/test.py       │
                 │          -> the student's submitted code                  │
                 │    Python libs: whatever is baked into THIS image         │
                 └──────────────────────────────────────────────────────────┘

             (workspace question)          student clicks "Open workspace"
                                         ▼
                 ┌──────────────────────────────────────────────────────────┐
                 │ 3. workspaceOptions.image  (a workspace container)       │
                 │    e.g. ucsbcs/workspace-jupyterlab-python-datascience    │
                 │    runs: JupyterLab / VS Code, and the student's notebook │
                 │    Python libs: whatever is baked into THIS image         │
                 └──────────────────────────────────────────────────────────┘
```

| Stage | What runs there | Where `datascience` comes from | Maintained in |
|---|---|---|---|
| 1. PL server | `server.py` (`generate`, `grade`, …), custom elements | Vendored into `serverFilesCourse/` in the **course repo** (PL adds that dir to `sys.path`) | the course repo, e.g. `pl-ucsb-cmpsc5ab/install-datascience.sh` |
| 2. External grader | `tests/test.py` + student code | Baked into `ucsbcs/grader-python-datascience` | **this repo** |
| 3. Workspace | JupyterLab, student notebooks | Baked into `ucsbcs/workspace-jupyterlab-python-datascience` | **this repo** |

Stage 1 cannot be changed with a Docker image (PrairieLearn controls that
container), which is why the course repo vendors a copy into
`serverFilesCourse/`. Stages 2 and 3 can only be changed with a Docker image, which
is what this repo is for.

A typical externally graded `datascience` question touches stages 1 and 2:
`server.py` generates the variant and often builds a reference `Table`; the
grader container runs the student's code and compares. **Both** need `datascience`,
and they should be the **same version**, otherwise the reference answer and the
student's answer can legitimately differ.

# Symptoms: what it looks like when `datascience` is missing somewhere

| Symptom | Which stage is broken | Fix |
|---|---|---|
| Question page shows a red traceback ending in `ModuleNotFoundError: No module named 'datascience'`, with `server.py` in the stack trace | 1 — PL server | `datascience` is not in `serverFilesCourse/` of the course repo. Run the course repo's install script and commit the result. |
| Same, but the missing module is a *dependency* (`No module named 'IPython'`, `'folium'`, `'plotly'`, `'branca'`, …) and `serverFilesCourse/datascience/...` is in the trace | 1 — PL server | The vendored copy is incomplete. `datascience` imports IPython, folium, plotly and branca unconditionally; those (and their transitive deps) must also be in `serverFilesCourse/`. |
| Question renders fine, but on submit the grading result panel shows `ModuleNotFoundError: No module named 'datascience'` with `tests/test.py` or the student file in the trace | 2 — grader | The question's `externalGradingOptions.image` is a plain `prairielearn/grader-python` (or some other image without `datascience`). Point it at `ucsbcs/grader-python-datascience:latest`. |
| On submit: `OCI runtime create failed: ... exec: "/python_autograder/run.sh": stat /python_autograder/run.sh: no such file or directory` | 2 — grader, wrong kind of image | The question is using a **workspace** image (or some other non-grader image) as its grader. Switch `externalGradingOptions.image` to `ucsbcs/grader-python-datascience:latest`. |
| On submit: grading "pending" forever, or an error about the image not being found / not synced | 2 — grader, image not synced | Production PrairieLearn only runs images that have been synced. Do the [sync step](#crucial-prairielearn-step-syncing-images). |
| Workspace opens, but `import datascience` in a notebook fails | 3 — workspace | `workspaceOptions.image` is the plain `prairielearn/workspace-jupyterlab-python`. Switch to `ucsbcs/workspace-jupyterlab-python-datascience:latest`, then sync. |
| Workspace fails to launch / image not found | 3 — workspace, image not synced | Sync the image (see below). |
| Everything "works", but a student's correct answer is marked wrong, or a reference solution produces different output in the grader than in `server.py` | 1 vs 2 version skew | The `serverFilesCourse/` copy and the grader image have different `datascience` versions. See the update procedure below; bump both together. |
| Images on Docker Hub haven't been rebuilt in months even though the weekly schedule exists | this repo's CI | GitHub disables scheduled workflows after 60 days with no commits. Go to **Actions → 10 - build-and-push-both → Enable workflow**, or just push a commit. |

To check which `datascience` version is actually in an image:

```
docker run --rm ucsbcs/grader-python-datascience:latest python -c "import datascience; print(datascience.__version__)"
docker run --rm ucsbcs/workspace-jupyterlab-python-datascience:latest /opt/conda/bin/python -c "import datascience; print(datascience.__version__)"
```

And in the course repo: `cat serverFilesCourse/datascience/version.py`.

# Why the `datascience` version is pinned

The Dockerfiles install `datascience==<version>`, not just `datascience`, and the
version lives in **one** file, [`datascience-version.txt`](datascience-version.txt),
that the workflows pass into every image.

Reasons:

1. **Consistency across stages.** The grader (stage 2), the workspace (stage 3), and
   the course repo's `serverFilesCourse/` (stage 1) must all agree. An unpinned
   `pip install datascience` in the Dockerfile means the version changes silently
   whenever the weekly rebuild happens to run after a new PyPI release, while the
   course repo's vendored copy does not. That is exactly the kind of skew that
   produces "my answer is right but it's marked wrong" tickets mid-quarter.
2. **Reproducible grading.** An exam graded on Monday should grade identically if
   regraded on Friday. The base `prairielearn/*` images still float (that is
   intentional, PL ships security and autograder fixes that way), but the library
   whose behaviour the questions actually depend on does not.
3. **Deliberate upgrades.** Upgrading becomes a reviewed PR with a diff, a CI build,
   and a date, rather than something that happened in a scheduled job nobody watched.
4. **Version tags on Docker Hub.** Because the version is known at build time, each
   image is also pushed as `ucsbcs/<image>:<version>`. A question can pin to that tag
   if it ever needs to be insulated from an upgrade.

# How to update `datascience` everywhere

Do this when a new `datascience` release is needed. Do it **between** assessments,
not during one.

1. **Pick the version** on PyPI: <https://pypi.org/project/datascience/#history>.

2. **In this repo**, open a PR that changes **one line**:

   ```
   echo "0.18.2" > datascience-version.txt
   ```

   Optionally, also bump the `ARG DATASCIENCE_VERSION=` default in each
   `images/*/Dockerfile` so manual local builds match. (The workflows override
   it, so this is cosmetic, but it avoids confusion.)

   Merging to `main` triggers `10-build-and-push-both.yml` automatically
   (it has a `push` trigger on `datascience-version.txt`). Check the Actions tab
   for a green run; both images are then on Docker Hub with `:latest` and
   `:<version>` tags.

3. **In PrairieLearn**, sync the images (see [below](#crucial-prairielearn-step-syncing-images)).
   Until you do, production PL keeps running the old image.

4. **In each course repo that vendors `datascience` into `serverFilesCourse/`**
   (e.g. `pl-ucsb-cmpsc5ab`), update the pinned version in its install script
   (`install-datascience.sh`, the `datascience==...` entry), re-run the script,
   and commit the resulting changes under `serverFilesCourse/`. Then sync the course
   in PL as usual.

   Note that the course repo vendors `datascience` *and the dependencies that the
   PrairieLearn server image lacks* (IPython, folium, plotly, branca and their
   transitive deps), because the server image cannot be customised. If a new
   `datascience` release adds a dependency, the course's install script needs that
   added to its package list too; the symptom is a `ModuleNotFoundError` for the new
   package in `server.py` (stage 1 in the table above).

5. **Verify** with the `docker run ... print(datascience.__version__)` commands
   above, and by opening one `datascience` question of each kind (externally
   graded, workspace) in PL and submitting an answer.

# How the images get built

Three workflows live in `.github/workflows/`:

| Workflow | Triggers | Builds |
|---|---|---|
| `10-build-and-push-both.yml` | push to `main` touching `images/**`, `datascience-version.txt` or the workflows; weekly on Sunday 00:00 UTC; manual | both images |
| `12-build-and-push-grader-python-datascience.yml` | manual only | grader image |
| `14-build-and-push-workspace-jupyterlab-python-datascience.yml` | manual only | workspace image |

The weekly schedule exists so the images pick up updates to the upstream
`prairielearn/*` base images even when nothing changes here. **Be aware that GitHub
disables scheduled workflows in any repo with no commits for 60 days**, and does
not notify anyone. If Docker Hub shows the images haven't been pushed in a while,
go to Actions → the workflow → "Enable workflow", or push any commit.

Images are built for `linux/amd64` and `linux/arm64/v8`. Production PrairieLearn
uses amd64; arm64 is for local development on Apple Silicon.

The images can also be built by hand, e.g.:

```
cd images/grader-python-datascience
docker login -u ucsbcs
docker build --platform linux/amd64 \
  --build-arg DATASCIENCE_VERSION=$(cat ../../datascience-version.txt) \
  -t ucsbcs/grader-python-datascience:latest .
docker push ucsbcs/grader-python-datascience:latest
```

The Docker Hub credentials are GitHub Actions secrets (`DOCKER_USERID` variable and
`DOCKER_PAT` secret), so it is safe for this repo to be public. That way the
Actions minutes are charged against the much larger budget for public repos.

# Crucial PrairieLearn step: syncing images

Production PrairieLearn does **not** pull `:latest` on every grading job. It runs
the copy of the image it last synced. So after any rebuild you need to:

* Navigate to your course page on PrairieLearn
* Go to the **Sync** tab
* Find the **Docker images** section
* Click **Sync** next to the image tags your questions use, so PrairieLearn pulls
  down the fresh builds.

<img width="1044" height="528" alt="image" src="https://github.com/user-attachments/assets/ccf55366-bbfb-45d1-b828-dcf483995386" />

Or, use the "Sync all images from Dockerhub to PrairieLearn" button:

<img width="419" height="57" alt="image" src="https://github.com/user-attachments/assets/cb44e8ea-8a05-4c80-afe4-31d0158e4ee1" />

(When running PrairieLearn locally in Docker, this step is not needed; local PL
pulls images directly.)
