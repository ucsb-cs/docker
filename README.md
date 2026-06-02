# docker

Github Actions Scripts to generate Docker Images under ucsbcs docker account


# PrairieLearn datascience images

For CMPSC 5A and CMPSC 5B, the Python module `datascience` is crucial to the course
material, and it is not installed by default in PrairieLearn.

In order to use `datascience` in external graders, and workspace questions,
we need to build custom docker images that:

* start with the images created by the PrairieLearn team
* add the `datascience` library

In the folder `images`, you will find a subdirectory for each of these
libraries.    Under each of those folders is a Dockerfile that does exactly this.

For example, under `grader-python-datascience`, we find this `Dockerfile`:

```dockerfile
# Use the original PrairieLearn Python grader image as the base
FROM prairielearn/grader-python:latest

# Install the datascience package
RUN pip install --no-cache-dir datascience
```

This is typically run at the command line with something like this:

```
docker login -u ucsbcs
docker build --platform linux/amd64 -t ucsbcs/grader-python-datascience:latest .
docker push ucsbcs/grader-python-datascience:latest
```

However, these commands can be run automatically by the github actions script 
`10-build-and-push.yml`, which can be run on demand, or scheduled to run periodically
so that the images stay up-to-date with those from PrairieLearn.

The secrets are protected so it is safe to make this repo public.  That way, the minutes used to run the Github Action script are charged against the larger budget of "public" github actions scripts minutes,rather than the much more restrictive budget of private repo minutes.

## Crucial PrairieLearn Step

In order to be sure that you are getting the latest images, you need to periodically:

* Navigate to your course page on PrairieLearn
* Go to the Sync tab
* Find the Docker images section
* Click Sync next to your new image tags so PrairieLearn pulls down your fresh builds.


