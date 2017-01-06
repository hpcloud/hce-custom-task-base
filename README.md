## Helion Cloud Engine - Custom Tasks
___

#### Description

Custom Tasks can be used to replace the build step, replace the test step, and add a variable number of post-deploy steps to a Helion Code Engine pipeline. 

These custom tasks allow the developer to add custom behavior such as updating a database or add support for a new language as seen here: [Doge-Weather](https://github.com/gregarcara/doge-weather). Post-deploy tasks allow adding additional steps such as running functional tests against your deployed application.

#### Usage

To add custom tasks to your project, you need to create a folder inside your project repository called hce-ci where the custom steps will live. For each step you want to customize you'll need to include a yml file with the following names

Build task: `build.yml`

Test task: `test.yml`

Post-deploy tasks: `post-deploy\*.yml` (where \* is any valid file characeters).

Post-deploy tasks are run in alphabetical order, so naming them post-deploy-1-funcitonal-test.yml, post-deploy-2-validate-url.yml could be a good practice.

Under your hce-ci folder, you should also have a folder for your scripts/binaries. Make sure all scripts and binaries are executable. For this example we will use the name `scripts`. So you might have a structure that looks like the following:

```
.
+-- build.yml
+-- test.yml
+-- post-deploy-1-validate-url.yml
+-- post-deploy-2-functional-tests.yml
+-- scripts
|   +-- build
|   +-- test
|   +-- validate-url
|   +-- functional-tests
```


##### Custom Build Tasks
When creating a custom build task start with the template below.

*build.yml*
```
---
platform: linux

image_resource:
  type: docker-image
  source:
    repository: crystallang/crystal
    tag: 0.20.1

inputs:
  - name: helionce-git-repo
  - name: custom-task-base
outputs:
  - name: build_code_output

run:
  path: sh
  args:
  - -exc
  - helionce-git-repo/hce-ci/scripts/build helionce-git-repo build_code_output
```

You can select a docker repostory and tag you want the build step to use. If you don't specify a tag `latest` will be used. It is required that your docker image has bash installed, otherwise, you can use any image. The inputs and outputs are manditory and should not be changed.

You can see in the run section, your script is passed your repo sources and the output directory. These two parameters are required. Additional parameters could be passed. In most cases, you only need to change the repository and tag of the docker image.

Below you can find an example build script.

*scripts/build*
```
#!/bin/bash

source "custom-task-base/task-base"

task() {
  cd "$source_dir"
  crystal deps
  crystal build --release src/doge-weather.cr
}

main "$@"
```

The script custom-task-base/task-base will capture logs, run time, move your build code to the output directory, and other tasks required by Helion Code Engine. You should only edit lines in the `task` function. The first line in the task function moves you into your project sources. After that, you can run whatever commands are required to build your project. For example, in a node project you may run:

```
npm install
npm build
```

You could also add commands to do tasks such as database schema updates here.

##### Custom Test Tasks

Custom test tasks run almost identitically to the build step. The main difference is in the boilerplace test.yml code:

*test.yml*
```
---
platform: linux

image_resource:
  type: docker-image
  source:
    repository: crystallang/crystal
    tag: 0.20.1

inputs:
  - name: custom-task-base
  - name: helionce-git-repo
  - name: build_code_output
outputs:
  - name: test_code_output

run:
  path: sh
  args:
  - -exc
  - helionce-git-repo/hce-ci/scripts/test build_code_output test_code_output
```

This step takes an additional input (the output of your build step) and outputs a new folder with the results of your test step. Otherwise, the steps are the same, passing in the output of your build step and and output folder name to be used by the Helion Code Engine script. You may use a different docker image repository and tag than your build step if required. After that, the test script is very similar as well.

*scripts/test*
```
#!/bin/bash

source "custom-task-base/task-base"

task() {
  cd "$source_dir/app"
  crystal spec
}

main "$@"
```

This looks very similar to the build script above. The biggest difference is the locaiton of our code is now in the `$source_dir/app` folder. Helion Code Engine moves the project code to the app folder after the build step so it can store other artifacts along side the application (the afore mentioned log files and run times). The only line you would need to change here would be the `crystal spec` line. For node you would write something similar to:
```
npm test
```

##### Post Deploy Tasks

Custom post-deploy tasks are again, very similar to the custom build and test tasks, only varying in the inputs and outputs provided. Here is an example post-deploy yml file:

*post-deploy-1.yml*
```
---
platform: linux

image_resource:
  type: docker-image
  source:
    repository: ubuntu

inputs:
  - name: custom-task-base
  - name: helionce-git-repo
  - name: build_code_output
outputs:
  - name: post-deploy-1_output

run:
  path: sh
  args:
  - -exc
  - helionce-git-repo/hce-ci/scripts/post-deploy-1 build_code_output post-deploy-1_output
```

This should look familiar by now, the only thing to note here is the name of the output folder, which should be the name of your yml file without the `.yml` and appending `_output`. So if your file was `post-deploy-1-test-url.yml` then your output folder should be `post-deploy-1-test-url_output` both in the outputs and the run section. Make note of your outputs folder name. If you have multiple post-deploy tasks you can add the folder from a previous step as an input and use it in your script if you require it.

The script for the post-deploy task is similar to both the build and test step above:

*scripts/post-deploy-1*
```
#!/bin/bash

source "custom-task-base/task-base"

task() {
  echo 'Post Deploy 1'
  echo $APPLICATION_URL
}

main "$@"
```

The only new thing here is that there is an environment variable available which holds the URL of your deployed application. This is useful if you want to run a curl command to make sure your web application is running, or run a full suite of functional tests against it.
