title: Continuous Deployment with Yeoman and Jenkins
author: Pascal Hartig
date: 2013-07-26
authorrank: https://plus.google.com/102129173185887281562
summary: We’ve been thinking a lot about deployment at the Yeoman team recently. A well-optimized build process is only half the story, because in the end you certainly want to share your work with someone else. The project I’m currently working on at weluse uses AngularJS, Yeoman and we came up with a Continuous Deployment setup I’m really happy with, that I would like to share with you.

<img src="//i.imgur.com/WAKSD3m.jpg" alt="" style="max-width: 100%">

We’ve been thinking a lot [about deployment](http://addyosmani.com/blog/making-maven-grunt/) at the [Yeoman team](http://yeoman.io) recently. A well-optimized build process is only half the story, because in the end you certainly want to share your work with someone else. The project I’m currently working on at weluse uses AngularJS, Yeoman and we came up with a Continuous Deployment setup I’m really happy with, that I would like to share with you.

Whenever a new commit is pushed to the master branch, our internal [Jenkins](http://jenkins-ci.org/) server picks it up and runs not only the test suite, but the complete grunt pipeline. It also installs all the bower components before that, as those are not part of our repository. There are conflicting opinions on whether committing your components to the repository is a good idea or not. In our case we haven’t seen any negative side-effects. If the build and test suite passes, Jenkins updates or initializes a new git repository in the `dist` folder that the grunt build created and pushes the **complete content** to a dedicated deployment remote. After that, Jenkins kicks off capistrano which checks out the deployment remote on the staging server.

## Setting up Jenkins

[<img style="float: right; margin: 1em;" src="//i.imgur.com/YawZ3Bf.png" alt="Jenkins Setup" width="400px">](//i.imgur.com/YawZ3Bf.png)

Since Yeoman and Jenkins are two fine gentlemen, they have no problem
cooperating with each other.  I'm using the term Yeoman here in the sense of a
project setup with Bower and a certain Grunt task layout, which includes a `build`
and `test` task as well as a seperate `dist` directory for the build artifacts.
You can integrate this setup into any Grunt project, though, with a little bit
of additional work.

Given a project layout like this the setup is very straightforward if you have a recent installation of node.js and npm on the CI server. Just create a new empty project and configure it to check out the desired branches of your repository. The interesting part is the build step. In an "Execute Shell" task, add a shell script like this:

```bash
export npm_config_prefix=.npm/
export PATH=.npm/bin:$PATH
npm install -g bower grunt-cli
bower install
npm install
grunt test
```

Notice that we overrode the npm prefix here to keep the `grunt` and `bower` packages limited to the current workspace.

The "Post Build Task" is even less spectacular and simply executes `grunt deploy` if the previous step was successful. This, however, is something you don’t get out of the box with Yeoman (yet).

## Setting up grunt

While the previous part was rather generic, the deployment part can be something that can be very specific to the needs and requirements of the environment you’re working in. In our case, the staging server is nothing but a dumb Ubuntu box provisioned with [nginx](http://nginx.org/) to serve everything that sits under a certain webroot as static files.

The first step of getting the build fragments to the webroot is pushing the `dist` folder to a custom remote. For that, we have this rather clunky shell script on the same level as the `Gruntfile.js` called `prepare_deploy.sh`:

```bash

#!/bin/sh

set -ex

cd dist/
git init || true
git remote add origin git@somegithost.com:deployment-repo.git || true
git fetch origin
git reset --mixed origin/master
git add -A
git commit -m "Deployment update" --allow-empty
git push origin master
cd ..
```

The benefit of this approach is that every deployment update represents its own commit and differences on the staging system can be reconstructed by comparing commits from the git history. This includes differences in bower components, which can be hard to detect otherwise.

For the Gruntfile, we make use of Sindre Sorhus’ excellent [grunt-shell](https://github.com/sindresorhus/grunt-shell) task to combine the calls to the shell script and capistrano into one handy `grunt deploy`:

```javascript

module.exports = function (grunt) {
  grunt.initConfig({
    ...
    shell: {
      options: {
        stdout: true,
        stderr: true,
        failOnError: true
      },
      pushdeploy: {
        command: 'sh prepare_deploy.sh'
      },
      cap: {
        command: 'cap staging deploy'
      }
    }
  });

  grunt.registerTask('deploy', [
    'build',
    'shell:pushdeploy',
    'shell:cap'
  ]);
};

```

We are not too happy with [Capistrano](http://www.capistranorb.com/) at the moment, because it has a severe lack of documentation for the latest version. However, for this simple case it works well enough.

Simply install it with `gem install capistrano`, run `cap init` and adjust the files in `config/` to your match your server configuration. Capistrano then takes care of checking out the repository, keeping track of previous deployments and allowing for an easy rollback in case that something broke.

## Improvements

The process obviously only works so well because of the simplicity of the application. That, however, is no accident, but by design. The Yeoman-based frontend application is just one part of a larger system, that includes several rails apps and a central RESTful API.

The ugliest piece of this process is the shell script to update the deployment repository, which I would love to find a nicer solution for. That said, just pushing to a branch and seeing the live application update just a couple of minutes later is a lot of fun and works incredibly well when you work directly with your customers.

You can [follow me on Twitter](http://passy.me/@) if you want to read more
about Yeoman and JavaScript tooling.
