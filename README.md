# Docker jenkins How-To

----
## Setting up docker jenkins
> this assumes you have already install docker

pull and run image from dockerhub
 
> connects port 8080 & 5050 from localhost to docker container.  
This will also start the initial jenkins setup

    $ docker run -p 8080:8080 -p 5050:5050 jenkins

optionally pull image first

    $ docker pull jenkins


## Setup jenkins

Now you will need to navigate to your local jenkins and Unlock it

> this requires the password autogenerated from your docker run.  
can also be found under: /var/jenkins_home/secrets/initialAdminPassword

[localhost-jenkins](http://localhost:8080/)

Next you want to *install suggested plugins*

` this will take a few minutes`

Next you want to create the first admin user.  After this you’ll be ready to start using jenkins

> user: admin  
Passwd: f00b@r1  
Email: <your@email.tld>

----
## My first jenkins job builder (*JJB*)

### Installing Jenkins Job Builder
we’ll first need to install jenkins jobs builder which requires python & pip

We will do it using *virtualenv* 

> **virtualenv** is a tool to create isolated Python environments. virtualenv creates a folder which contains all the necessary executables to use the packages that a Python project would need.

    $ virtualenv -p python2.7  jjb-env # -p use specific python version mirror your jenkins version
    $ . ~/jjb-env/bin/activate
    $ pip install jenkins-job-builder # if not using virtualenv use *--user*

#### JJB Configuration

JJB requires a configuration file to function. It searches for a configuration
file in the following locations, in order:

1. `~/.config/jenkins_jobs/jenkins_jobs.ini`
2. `/etc/jenkins_jobs/jenkins_jobs.ini`

For now we will create a JJB configuration file in $HOME.

    cat >> $HOME/jjb-test.ini <<EOF
    [jenkins]    
    user=admin  
    password=f00b@r1  
    url=http://localhost:8080  
    query_plugins_info=False   
    
    [job_builder]  
    include_path=ci/jjb/scripts  
    EOF

### JJB job definitions

At a high level we’ll cover some of the most basic definitions you require to build jobs

* project
* job-template
* macros (ie: builders)
* job
  * job-groups
* defaults

##### project
The purpose of a project is to collect related jobs together, and provide values for the variables in a Job Template. It looks like this:

    - project:
        name: my-kool-project
        jobs:
          - '{name}-unit-tests'
    
Any number of arbitrarily named additional fields may be specified, and they will be available for variable substitution in the job template. Any job templates listed under jobs: will be realized with those values. The example above would create the job called ‘my-kool-project-unit-tests’ in Jenkins. 

The jobs: list can also allow for specifying job-specific substitutions as follows:

    - project:
        name: my-kool-project
        jobs:
          - '{name}-unit-tests':
              mail-to: developer@nowhere.net
          - '{name}-perf-tests':
              mail-to: projmanager@nowhere.net


The following example would create multiple jobs from the list of `pyver`

    - job-template:
        name: '{name}-{pyver}'
        builders:
          - shell: 'git co {branch_name}'
    
    - project:
       name: project-name
       pyver:
        - 26:
           branch_name: old_branch
        - 27:
           branch_name: new_branch
       jobs:
        - '{name}-{pyver}'

#### job-template
A jobs template can be reuse for multiple jobs that are similar using variables.  Use a Project to realize the job with appropriate variable substitution. Any variables not specified at the project level will be inherited from the Defaults.

Basic usage (*It’s name will depend on what is supplied to the Project.*):

    - job-template:
        name: '{name}-unit-tests'

If you use the variable {template-name}, the name of the template itself (e.g. {name}-unit-tests in the above example) will be substituted in. This is useful in cases where you need to trace a job back to its template.

To facilitate reuse of templates with many variables that can be substituted, but where in most cases the same or no value is needed, it is possible to specify defaults for the variables within the templates themselves.

This can be used to provide common settings for particular templates. Notice the `Ids` use to simplify mutliple jobs with same template.  For example:

```yaml
    - project:
        name: template_variable_defaults
        jobs:
            - 'template-variable-defaults-{num}':
                num: 1
                disabled_var: true
            - 'template-variable-defaults-{num}':
                test_var: Goodbye World
                num: 2

    - job-template:
        # template specific defaults
        # empty value causes disabled_var to be ignored internally
        disabled_var:
        test_var: Hello World
        type: periodic
    
        # template settings
        name: 'template-variable-defaults-{num}-{type}'
        id: 'template-variable-defaults-{num}'
        disabled: '{obj:disabled_var}'
        builders:
          - shell: |
             echo "Job Name: template-variable-defaults-{num}-{type}"
             echo "Variable: {test_var}"
```

#### macros
These are thing like builders ie:

    # this generic macro builders that takes argument *yourname"
    - builder:
        name: hello-world
        builders:
         - shell: "echo hello world {yourname}"

    # static builder using a predefined generic parameter macro
    - builder:
        name: hello-world-flynn
        builders:
         - hello-world:
            myname: "flynn"

Using it in a job
    
    - job:
        name: "testingjob"
        builders:
         # The static macro:
         - hello-world-flynn
         # Generic macro call with a parameter
         - hello-world:
            myname: "ZERO"

#### job
job is where you define your job ideally using job-template and macros

Lets first look at a static job

    - job:
        name: job-name
        project-type: freestyle
        defaults: global
        description: 'Do not edit this job through the web!'
        disabled: false
        display-name: 'Fancy job name'
        concurrent: true
        workspace: /srv/build-area/job-name
        quiet-period: 5
        block-downstream: false
        block-upstream: false
        retry-count: 3
        node: NodeLabel1 || NodeLabel2 || cucumber || trusty
        logrotate:
          daysToKeep: 3
          numToKeep: 20
          artifactDaysToKeep: -1
          artifactNumToKeep: -1


##### job parameters definition
* ***project-type:*** Defaults to “freestyle”, but “maven” as well as “multijob”, “flow”, “pipeline” or “externaljob” can also be specified.
* ***defaults:*** Specifies a set of Defaults to use for this job, defaults to ‘’global’‘. If you have values that are common to all of your jobs, create a global Defaults object to hold them, and no further configuration of individual jobs is necessary. If some jobs should not use the global defaults, use this field to specify a different set of defaults.
* ***description:*** The description for the job. By default, the description “!– Managed by Jenkins Job Builder” is applied.
* ***disabled:*** Boolean value to set whether or not this job should be disabled in Jenkins. Defaults to false (job will be enabled).
* ***display-name:*** Optional name shown for the project throughout the Jenkins web GUI in place of the actual job name. The jenkins_jobs tool cannot fully remove this trait once it is set, so use caution when setting it. Setting it to the same string as the job’s name is an effective un-set workaround. Alternately, the field can be cleared manually using the Jenkins web interface.
* ***concurrent:*** Boolean value to set whether or not Jenkins can run this job concurrently. Defaults to false.
* ***workspace:*** Path for a custom workspace. Defaults to Jenkins default configuration.
* ***folder:*** The folder attribute provides an alternative to using ‘<path>/<name>’ as the job name to specify which Jenkins folder to upload the job to. Requires the CloudBees Folders Plugin.
* ***child-workspace:*** Path for a child custom workspace. Defaults to Jenkins default configuration. This parameter is only valid for matrix type jobs.
* ***quiet-period:*** Number of seconds to wait between consecutive runs of this job. Defaults to 0.
* ***block-downstream:*** Boolean value to set whether or not this job must block while downstream jobs are running. Downstream jobs are determined transitively. Defaults to false.
* ***block-upstream:*** Boolean value to set whether or not this job must block while upstream jobs are running. Upstream jobs are determined transitively. Defaults to false.
* ***auth-token:*** Specifies an authentication token that allows new builds to be triggered by accessing a special predefined URL. Only those who know the token will be able to trigger builds remotely.
* ***retry-count:*** If a build fails to checkout from the repository, Jenkins will retry the specified number of times before giving up.
* ***node:*** Restrict where this job can be run. If there is a group of machines that the job can be built on, you can specify that label as the node to tie on, which will cause Jenkins to build the job on any of the machines with that label. For matrix projects, this parameter will only restrict where the parent job will run.
* ***logrotate:*** The Logrotate section allows you to automatically remove old build history. It adds the logrotate attribute to the Job definition. All logrotate attributes default to “-1” (keep forever). Deprecated on jenkins >=1.637: use the build-discarder property instead
* ***jdk:*** The name of the jdk to use
* ***raw:*** If present, this section should contain a single xml entry. This XML will be inserted at the top-level of the Job definition.


##### defaults
Defaults collect job attributes (including actions) and will supply those values when the job is created, unless superseded by a value in the ‘Job’_ definition. If a set of Defaults is specified with the name global, that will be used by all Job (and Job Template) definitions unless they specify a different Default object with the defaults attribute. For example:

    - defaults:
        name: global
        description: 'Do not edit this job through the web!'

Will set the job description for every job created.

You can define variables that will be realized in a Job Template.

    - defaults:
        name: global
        arch: 'i386'
    
    - project:
        name: project-name
        jobs:
            - 'build-{arch}'
            - 'build-{arch}':
                arch: 'amd64'
    
    - job-template:
        name: 'build-{arch}'
        builders:
            - shell: "echo Build arch {arch}."

Would create jobs build-i386 and build-amd64.

You can also reference a variable {template-name} in any value and it will be substituted by the name of the current job template being processed.



### Creating my first job
For our testing purposes we’ll download a predefined job with git clone

    git clone git@github.com:pulp/pulp-ci.git

Lets test our job before we do anything else (-o for especifing output file)

    cd ci/jjb
    Jenkins-jobs --conf $HOME/jjb-test.ini test -o /path/to/outdir/ ci/jjb/jobs hello-world

Now lets add our job

    jenkins-jobs --conf $HOME/jjb-test.ini --ignore-cache update ci/jjb/jobs hello-world

You should see output similar to this one:

    INFO:root:Updating jobs in ['ci/jobs'] (['hello-world'])
    ... output omitted ...
    INFO:jenkins_jobs.builder:Number of jobs generated:  1
    INFO:jenkins_jobs.builder:Creating jenkins job pulp-installer
    INFO:root:Number of jobs updated: 1

Now you should be able to see the job in jenkins and are ready for creating NEW jobs
