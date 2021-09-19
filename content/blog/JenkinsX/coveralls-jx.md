---
title: Integrating JenkinsX with coveralls
tags: ["JenkinsX", "coveralls", "testing", "v3"]
date: '2021-09-19'
---

Recently at work, we wanted to have test coverage for our codebase.
Generating code coverage is great, but how do we make that coverage report accessible to every one in the engineering group and ensure no new changes decrease the test coverage?

Enter [coveralls](https://coveralls.io/)!

Why coveralls?
* Support for github which we use for source code management
* Support for many programming languages including C++ and python
* Simple but rich UI to show lines which are covered or not covered by test cases
* Compare coverage with base branch for pull requests, which ensures that we do not merge a pull request which would decrease the code coverage for the repository

Unfortunately, there are no guides on how to send coverage data from JenkinsX to coveralls.
Coveralls has an API that can be used to send the coverage data to, so integrating JenkinsX with coveralls is not that hard.

Assumptions:
* You have a coveralls account, and generated a token for the private repository.
* You have generated test coverage report using native tools in the language of choice (`gcovr` for C++, `go test` for golang etc ...) in a format that can be understood by the coveralls API.

Assuming you have a git merge step in the pipeline, tweak it to not generate a git commit
```
- envFrom:
    - secretRef:
        name: jx-boot-job-env-vars
        optional: true
    image: ghcr.io/jenkins-x/jx-boot:3.2.174
    name: git-merge
    resources: {}
    script: |
    #!/usr/bin/env sh
    # Avoid creating a merge commit when merging the pull request changes with the HEAD of the main branch
    jx gitops git merge --merge-args=["--no-commit"]
    # Avoid creating a commit when .jx/variables is created
    jx gitops variables --commit=false
    jx gitops pr variables
```

Apart from the changes made to avoid creating a (CI only) commit, there is an additional command to add pull request environment variables to the `.jx/variables` file.
This is important to get the name of the git branch which will be used by coveralls web UI.

The list of environment variables available in JenkinsX pipeline can be found [here](https://jenkins-x.io/v3/develop/reference/variables/#environment-variables)
The ones we are interested in are:
* BUILD\_NUMBER
* PR\_HEAD\_REF (which is available only if you have `jx gitops pr variable` in your pipeline)
* BRANCH\_NAME (which returns the PR number in case of JenkinsX)

Additionally set the coveralls token as an environment variable COVERALLS\_REPO\_TOKEN (you can read it from a kubernetes secret).
Optionally, you can also set `CI_NAME` as an environment variable, so coveralls can display that the build was triggered by JenkinsX

The various packages used to integrate different CIs with coveralls do not have support for setting values for JenkinsX like `service_name` and `service_pull_request` (They do have support for other CIs like circleCI, gitlabCI, github actions etc ...).
In our case, `gcovr` set the value for branch to `HEAD`.
However, one can use jq (in case of json) or simple sed/awk linux utlity to set the correct values.

Add this snippet to the JenkinsX pipeline step where you have generated the test coverage report.
```
 jq --arg branch "$PR_HEAD_REF" \
    --arg service_name "$CI_NAME" \
    --arg service_pull_request "$BRANCH_NAME" \
    '.git.branch=$branch | .service_name=$service_name | .service_pull_request=$service_pull_request' \
    coveralls.json > coveralls.json.tmp \
    && mv coveralls.json.tmp coveralls.json
```

Once packages like gcovr and coveralls-ruby can generate coveralls file specific to JenkinsX, this substitution step will not be neccessary.

Finally, to send the report to coveralls, one can add this snippet to the pipeline:
```
curl -i -F "json_file=@coveralls.json" https://coveralls.io/api/v1/jobs; echo
```
It's important to also enable coveralls build for the release pipeline (or any pipeline which is run on the base branch) to generate a coverage report for the master branch.
This will ensure that when a pull request is opened, coveralls will show is the coverage has increased in the pull request compared to the base branch.
