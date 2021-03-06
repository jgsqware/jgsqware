+++
date = "2015-11-09T08:00:00+01:00"
title = "Deploy Hugo on Github user pages"
+++

This blog is deployed on [GitHub User Pages](https://help.github.com/articles/user-organization-and-project-pages/#user--organization-pages) with [Wercker](http://wercker.com/).

First, I've follow the [Hugo User Guide to deploy with Wercker](https://gohugo.io/tutorials/automated-deployments/). But it deploy on Github project pages and not Github User pages. You can use the Hugo user guide first than replace your `wercker.yml` file with the following one.

I'm planning to use Hugo as my personal blog generator. So, i've created a repository in Github [Rocket-Vision](https://github.com/juliengarcia/rocket-vision) that hold my **not build** blog.

Then I use Wercker to build and deploy on the [Github User Page repository](https://github.com/juliengarcia/juliengarcia.github.io).

## Configure Wercker file
To build and deploy on Wercker, you need a `wercker.yml` configuration file.

### Build steps
  I use the `arjen/hugo-build@1.6.1` step to build Hugo.

  I've defined the theme: *herring-cove* and decide to not publish the drafts

### Deploy steps
  First, I need to install **Git** to be able to push in the right repository. It's done by the `install-packages` step.

  Then, I need to push into the [juliengarcia.github.io](https://github.com/juliengarcia/juliengarcia.github.io) and not in the [Rocket-Vision](https://github.com/juliengarcia/rocket-vision). So, I use the `leipert/git-push@0.7.6` step.

  I configure the git token (`gh_oauth` param) and destination repository (`repo: juliengarcia/juliengarcia.github.io` param)

```yaml
box: debian
build:
  steps:
    - arjen/hugo-build@1.6.1:
      version: "0.14"
      theme: herring-cove
      flags: --buildDrafts=false

deploy:
  steps:
    - install-packages:
      packages: git
    - leipert/git-push@0.7.6:
      gh_oauth: $GIT_TOKEN
      repo: juliengarcia/juliengarcia.github.io
      basedir: public

```

Now, on every push on Rocket-Vision repository, the hugo website is build then deploy on the juliengarcia.github.io repository and available on [http://juliengarcia.github.io](http://juliengarcia.github.io)
