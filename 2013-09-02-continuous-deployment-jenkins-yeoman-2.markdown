title: Continuous Deployment with Yeoman and Jenkins Part II
author: Pascal Hartig
date: 2013-09-02
authorrank: https://plus.google.com/102129173185887281562
summary: The first part of this article inspired Frédéric Camblor to create a Jenkins plugin that simplifies the process a lot.

[The first part of this
article](https://weluse.de/blog/continuous-deployment-with-yeoman-and-jenkins.html)
generated some great feedback. Some of which critized that node.js
was too difficult to set up with Jenkins. This inspired the magnificient
[Frédéric Camblor](https://twitter.com/fcamblor) to work on a plugin for Jenkins
to simplify the process.

<center>
<blockquote class="twitter-tweet"><p>Working on nodejs <a
href="https://twitter.com/jenkinsci">@jenkinsci</a> plugin because I want to
simplify <a href="https://twitter.com/passy">@passy</a>&#39;s continuous
deployment with <a href="https://twitter.com/yeoman">@yeoman</a> / <a
href="https://twitter.com/bower">@bower</a> / <a
href="https://twitter.com/gruntjs">@gruntjs</a> :-)</p>&mdash; Frédéric Camblor
(@fcamblor) <a
href="https://twitter.com/fcamblor/statuses/361118719897903105">July 27,
2013</a></blockquote> <script async src="//platform.twitter.com/widgets.js"
charset="utf-8"></script>
</center>

The [NodeJS Plugin for
Jenkins](https://wiki.jenkins-ci.org/display/JENKINS/NodeJS+Plugin) is now
available in version 0.2 and counts almost 500 installations and can be directly
installed from Jenkins’ plugin manager with a single click.

After you installed the plugin, you can add one or more NodeJS installations in
the global system configuration for Jenkins, which allows you to define which
version of node you want to install and which global packages it should provide
out of the box. For Yeoman setups, you can use this to install `bower` and
`grunt-cli` so you don’t have to do this within your project configuration. It's
recommended that you fix the package versions of the global dependencies so you
don't get an unexpected build failure because of an update. You can specify
those like `grunt-cli@~0.1.0 bower@~1.2.0`.

![Setting up a node installation in the System Configuration](//i.imgur.com/msFjl85.png)
While we don’t require multiple NodeJS installation for our Yeoman needs, it’s
obvious how handy having several isolated node environments in different
versions is to test libraries against.

Having taken care of the global NodeJS config, we can adjust the project
configuration and dramatically reduce the amount of boilerplate code to run our
tests. In the **Build Environment** configuration section, you select the
previously created NodeJS installation, so you have the correct environment on
your `$PATH`. This enables you to cut the shell script in the **Build** section
down to three lines:

```bash
bower install
npm install
grunt test
```

![Build setup in the project configuration](https://i.imgur.com/hVODBkP.png)

And that’s it. No SSH-ing into your CI server to install and update the NPM
environments and no manual fiddling around with `$PATH` and npm prefixes. Thanks
Frédéric for making our lifes easier!
