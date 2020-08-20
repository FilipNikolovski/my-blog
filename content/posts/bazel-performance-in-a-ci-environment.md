---
title: "Bazel Performance in a CI Environment"
date: 2020-03-05T22:19:10+02:00
draft: false 
categories:
  - DevOps
tags:
  - Bazel
  - Monorepo
  - CI
  - Gitlab
---

Lately I've been obsessing with the performance of [Bazel](https://bazel.build/) in our CI environment. We've been using this tool for quite some time now for our Golang monorepo and since the beginning of its creation it has grown quite a lot, so we've hit a couple of road blocks with our setup. We're using [Gitlab CI](https://docs.gitlab.com/ee/ci/) as our continuous integration environment and we host the runners on our own AWS instances.

As a first setup, we we're using the [docker executor](https://docs.gitlab.com/runner/executors/docker.html) and we had an image that had all the necessary tools to run our tasks, including Bazel of course.
As our repository grew larger, our build times we're starting to become greater and greater and I've searched high and low on the internet for a solution but to no avail.

The biggest issue was the caching of the results. Bazel is much faster when it can utilize the cache to only rebuild what is necessary. To use this in our CI environment, Bazel has a [`--disk-cache`](https://docs.bazel.build/versions/master/remote-caching.html#disk-cache) flag which can be used to specify a directory to be used by Bazel  as a remote cache. The `gitlab-ci.yaml` file looked something like this:

```yaml
cache:
  key: bazel_cache
  paths:
    - .cache/

build:
  stage: build
  script:
    - bazel build --disk-cache=$CI_PROJECT_DIR/.cache //...
```

There is one problem though - the Bazel caches are huge. Downloading and uploading the cache folder actually took most of the time when running a job in the pipeline. Each job took 5min to just simply download and then re-upload the cache after Bazel is done running the tasks.

To try and mitigate this issue, we tried using multiple runners that can spawn dynamically on demand, using the [autoscale configuration](https://docs.gitlab.com/runner/executors/docker.html), so we ended up having each job be run by a separate Bazel server and then upload the cache folder to S3. But this didn't help much, in fact it was actually slower by 5-10% than the previous configuration.

Many recommendations that I've read on blog posts and github issues, suggest using clean builds with no caching whatsoever since that way is usually faster, but I was determined not to throw away the cache, since it's one of the main points of using Bazel in the first place.

So this time, I decided to try with the [shell executor](https://docs.gitlab.com/runner/executors/docker.html). My strategy was to have one instance for the runner, which wasn't ephemeral as the previous ones which used Docker containers, but a long running one with 4 concurrent runners, one for each job.

The shell executor checks out the project code to a build dir, which does not change over time:
```
<working-directory>/builds/<short-token>/<concurrent-id>/<namespace>/<project-name>
```

And the caches are kept in another directory:

```
<working-directory>/cache/<namespace>/<project-name>
```

The idea at first was for the runner to copy the cache folder in the project, let Bazel do it's thing and then copy the contents back, which should be faster than syncing the files on S3. With this approach we may have shaved 1 or 2mins from each job, but it wasn't enough.

I started searching the Gitlab CI's documentation again, and realized that we can utilize the [`GIT_CLEAN_FLAGS`](https://docs.gitlab.com/ee/ci/large_repositories/#git-clean-flags) variable, which was introduced in the 11.10 version.

Before running a job, the runner first performs a `git clean`, which cleans the build directory of any files and folders that were created during the previous run. The `GIT_CLEAN_FLAGS` variable gives us the capabilities of controlling which folders `git` can exclude from removal between **subsequent runs**. This is perfect for our use case, since we have existing directories that we **re-use** for our builds.

I ended up avoiding the cache mechanism from Gitlab, and created a cache directory **inside** the build directory instead, which survived consecutive jobs. This way Bazel can keep using the cache while not having to transfer it back and forth between the two folders, which drastically saves time.

The final gitlab-ci.yaml file looked something like this:

```yaml
variables:
  GIT_CLEAN_FLAGS: -ffdx -e .cache/

build:
  script: bazel build --disk_cache=$CI_PROJECT_DIR/.cache //...
```

And it was a great success! We managed to decrease our build times from 20mins to just 1-2mins, and utilize the caching properly.

### TL;DR

- Use a **shell** executor and a long-running instance in order to re-use the build directories.
- Use the Bazel `--disk-cache` flag to specify a cache directory inside the project's build directory.
- Don't use the cache mechanism from Gitlab.
- Utilize the `GIT_CLEAN_FLAGS` variable, so that when running `git clean` on subsequent runs, it will ignore the cache directory.
- Profit.

That's it folks, I hope this post will be of some use to you, if you have questions feel free to shoot me an email.
