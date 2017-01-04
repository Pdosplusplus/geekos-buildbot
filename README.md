# Buildbot to Geekos

First created the enviroment

## Create the file ```requirements.txt``` with the following content:

>
```bash
Jinja2==2.8
MarkupSafe==0.23
SQLAlchemy==0.7.10
buildbot==0.8.14
buildbot-slave==0.8.14
chardet==2.3.0
migrate==0.3.8
python-dateutil==2.5.3 
python-debian==0.1.28
requests==2.4.3
service-identity==16.0.0
sqlalchemy-migrate==0.7.2
```

## Now create the virtualenv:


>
```bash
$ virtualenv buildbot-geekos
```

#### Activate


>
```bash
$ source buildbot-geekos/bin/activate
```


## Install the dependencies to run buildbot


>
```bash
$ pip install -r requirements.txt
```

## Create buildbot master

* 1 - To create the master execute:

>
```bash
$ buildbot create-master master
```

* 2 - Change name of file master.cfg.sample to master.cfg

>
```bash
$ mv master/master.cfg.sample master/master.cfg
```

* 3 - Now start it:

>
```bash
$ buildbot start master
```

## Create buildbot slave

* 1 - To create the slave execute:

>
```bash
$ buildslave create-slave BASEDIR MASTERHOST:PORT SLAVENAME PASSWORD
```
or 

>
```bash
$ buildslave create-slave builder localhost:9989 builder 11
```

* 2 - The file located in BASEDIR/info/admin should contain your name and email address. This is the buildslave admin address, and will be visible from the build status page (so you may wish to munge it a bit if address-harvesting spambots are a concern).

* 3 - The file located in BASEDIR/info/host should be filled with a brief description of the host: OS, version, memory size, CPU speed, versions of relevant libraries installed, and finally the version of the buildbot code which is running the buildslave.

* 4 - Init Slave:

>
```bash
$ buildslave start builder
```

* 5 - Restart the buildmaster to take the configuration

>
```bash
$ buildbot restart master
```

* 6 - Config buildslaves

#### In the file ```master/master.cfg``` config the slaves:

```python
c['slaves'] = [buildslave.BuildSlave("example-slave", "pass")]
```

#### For the slave created:

```python
c['slaves'] = [buildslave.BuildSlave("builder", "11")]
``` 

#### And restart buildbot

>
```bash
$ buildbot restart master
```

## Check BUILDBOT


#### In your browser check the url:

```bash
localhost:8010
``` 

## CONFIGURACION BASIC

To basic config created a lists to each component which will be explained

Add in the file ```master.cfg```:


```python
c = BuildmasterConfig = {}

c['status'] = []

c['slaves'] = []

c['change_source'] = []

c['schedulers'] = []

c['builders'] = []

c['buildbotURL'] = 'http://localhost:8010/'

c['protocols'] = {'pb': {'port': 9989}}

forced_builders = []

vcs = {}

vcs['branches'] = ['master', 'desarrollo']

vcs['dev'] = 'desarrollo'

vcs['time'] = 60

vcs['pollinterval'] = 60 
```

## Time to use BUILDBOT

When you config buildbot, should be clear about several concepts:

## Change Sources

Which create a Change object each time something is modified in the VC repository. Most ChangeSources listen for messages from a hook script of some sort. Some sources actively poll the repository on a regular basis. All Changes are fed to the Schedulers.

#### List to Change Sources:

>
```python
c['change_source'] = []
```

## Schedulers

Which decide when builds should be performed. They collect Changes into BuildRequests, which are then queued for delivery to Builders until a buildslave is available.

####  List to Schedulers:

>
```python
c['schedulers'] = []
``` 

## Builders

Which control exactly how each build is performed (with a series of BuildSteps, configured in a BuildFactory). Each Build is run on a single buildslave.

#### List to Builders:

>
```python
c['builders'] = []
```

## Create Change Sources

Here we do a git-based change_source, which will verify the changes submitted by the repository. The parameters passed are the following:

* Url.
* Name the change.
* Branch repository.
* Time interval for a poll.

Already with this simple configuration we create a change_source.

In this example uses a repo private

Into to the file ```master.cfg```:


```python
from buildbot.changes.gitpoller import GitPoller

c['change_source'].append(
        GitPoller(repourl="https://myusername:mypassword@github.com/myorgname/myreponame.git"
              project= "geekos-buildbot",
              branches=["master"],
              pollinterval=60)
            )
```

## Create Builder

1 - To create a builder we need some things, which are:

* Name to builder.
* factory with all steps.
* slavename.

#### First defined the steps to the build:

Into to file ```master.cfg```:


```python
from buildbot.steps.shell import SetPropertyFromCommand, ShellCommand
from buildbot.process.properties import WithProperties, Interpolate
from buildbot.process.factory import BuildFactory

package = "connect-vpn"
URL_REPO = "https://github.com/Pdosplusplus/connect-vpn"
CLONE_DIR = 'build'
BUILD_DIR = os.path.join(CLONE_DIR, package)

all_steps = [
    ShellCommand(command = 'rm -rf *'),
    ShellCommand(command = ['git', 'clone', URL_REPO], workdir=CLONE_DIR, haltOnFailure = True),
    ShellCommand(command = ['git', 'checkout', vcs['dev']], workdir=BUILD_DIR),
    SetPropertyFromCommand(command = "cat debian/changelog", extract_fn=parse_changelog, workdir=BUILD_DIR, haltOnFailure = True),
    SetPropertyFromCommand(command = "cat debian/control", extract_fn=parse_control, workdir=BUILD_DIR, haltOnFailure = True),
    ShellCommand(command = ['gbp', 'buildpackage', '--git-pbuilder', '--git-dist=jessie', '--git-debian-branch='+vcs['dev'],
        '--git-upstream-tree='+vcs['dev'], '--git-arch=i386', '-us', '-uc'], doStepIf=check_i386, workdir=BUILD_DIR, haltOnFailure = True),
    ShellCommand(command = ['gbp', 'buildpackage', '--git-pbuilder', '--git-dist=jessie', '--git-debian-branch='+vcs['dev'],
        '--git-upstream-tree='+vcs['dev'], '--git-arch=amd64', '-us', '-uc'], doStepIf=check_amd64, workdir=BUILD_DIR, haltOnFailure = True),
    ShellCommand(command = WithProperties("lintian %(package)s_%(version)s_%(architecture)s.changes"), workdir=CLONE_DIR, haltOnFailure = False),
    ShellCommand(command = 'rm -rf *', workdir=UPLOAD_DIR),
]
```


2 - Added the builder


```bash
from buildbot.config import BuilderConfig
from buildbot.process.factory import BuildFactory

c['builders'].append(
        BuilderConfig(
	            name=package, 
	            factory=BuildFactory(all_steps), 
	            slavenames='builder'
         	)
        )
```

## Create Schedulers

Now we create a Scheduler, he will be responsible the collect changes into Build Requests to after delivery to a Builders until a buildslave is available

The parameters are:

* Name.
* Changefilter with name project and branch.
* treeStableTimer.
* builderNames


```python
sbched = SingleBranchScheduler(
        name=package,
        change_filter = filter.ChangeFilter(project=package, 
                            branch='master'
                            ),
        treeStableTimer = 10,
        builderNames = [package])
```

#### Add Scheduler to all schedulers:


```python
c['schedulers'].append(sbched)
```

#### Also we can add the option to the builder to that can be forced his construction or execution, for this:


```python
forced_builders.append(package)
``` 

#### And now we created a ForceScheduler that contain all forced builders:


```python
forced_scheduler = ForceScheduler(name='Forzar', builderNames = forced_builders)
```

#### And we add the force_builders to the schedulers:


```python
c['schedulers'].append(forced_scheduler)
```

## Autentication

We will create a Object Authz with the configuration to access the buildbot into the file ```master.cfg```:


from buildbot.status import html
from buildbot.status.web import authz, auth


```python
authz_cfg=authz.Authz(
    # change any of these to True to enable; see the manual for more
    # options
    auth=auth.BasicAuth([("pyflakes","pyflakes")]),
    gracefulShutdown = False,
    forceBuild = 'auth', # use this to test your slave once it is set up
    forceAllBuilds = True,
    pingBuilder = True,
    stopBuild = True,
    stopAllBuilds = True,
    cancelPendingBuild = True,
)
```

## Web Status

We add to the WebStatus with the configuration saved in authz_cfg


```python
c['status'].append(html.WebStatus(http_port=8010, authz=authz_cfg))
```


## Functions Generic to build a debian package

And to finish we add the functions generic

Into the file ```master.cfg```:


```python

def parse_changelog(rc, stdout, stderr):
    '''
    Parsear changelog a partir de un cat <archivo-changelog>
    '''
    c = Changelog(stdout)
    return {'version' : c.full_version,
            'package' : c.package}

def parse_control(rc, stdout, stderr):
    '''
    Parsear un archivo debian control
    '''
    archi = ''

    for control in deb822.Deb822.iter_paragraphs(stdout):
        archi = control.get('Architecture')

    if archi == 'all' or archi == 'any':
        archi = 'i386' 

    return {'architecture' : archi}

def check_i386(step):
    '''
    Funcion que solamente chequea la arquitectura
    '''
    arch = step.build.getProperty('architecture')
    if arch  == 'i386' or arch == 'all' or arch == 'any':
        return True

    return False

def check_amd64(step):
    '''
    Funcion que solamente chequea la arquitectura
    '''
    if (step.build.getProperty('architecture') == 'amd64'):
        return True

    return False
```

##### Restart buildbot

```bash 
$ buildbot reconfig master
```

or

```bash
$ buildbot restart master
```