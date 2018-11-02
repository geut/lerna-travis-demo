# Playing with chan on monorepos
[![Build Status](https://travis-ci.com/geut/lerna-travis-demo.svg?branch=master)](https://travis-ci.com/geut/lerna-travis-demo)

> :warning: This is a demo project showcasing: lerna + chan + travis + npm

Okay, let's say you have a monorepo and you are happy with it. Everything seems to be OK but one day your monorepo open source project starts to be used by someone else so you say let's automatize things. 

Probably at this point, some steps are _manually_ controlled, like `publish` for example. At the same time, you are a cool person and make use of travis for CI. 

So you say to yourself, what about _CDing_? Can I release a new _lerna-managed_ package version from Travis? Is that even possible?

Ok, lets figure it out. Oh! But one more thing. I've mentioned that you are cool person, and that means your project will maintain a `CHANGELOG.md` file. OMG, that is so cool. Luckily for us, the GEUT team is here to help us, they have built this little tool called [chan](https://github.com/geut/chan) which can be used to easily create and **maintain** a changelog file based on [keepachangelog](https://keepachangelog.com/en/1.0.0/) directives. Which in the end, results in a human readable list of changes about the progress of our project.

## How?

I've found 2 ways to solve this problem:

**A.** [Fully "automatic"](#a---fully-automatic)   
**B.** [Half & Half](#b---half--half-star) 

Let's explore them.

### A - **Fully Automatic**

It's really cool to automate everything but there is a thing to solve, "how to tell travis what should be the next semver bump?". Anyway, this workflow can still be useuful for staging/dev scenarios where you can publish to an internal registry for example or something like that.

#### Setup

The important bits of your `.travis.yml`:

```yaml
language: node_js
node_js:
  - "10"
cache:
  yarn: true
  directories:
    - "node_modules"
before_install:
  - curl -o- -L https://yarnpkg.com/install.sh | bash -s
  - export PATH="$HOME/.yarn/bin:$PATH"
before_script:
  - yarn run lerna run prepublish
script:
  - yarn test
before_deploy:
  - git config --global user.email ${GITHUB_EMAIL}
  - git config --global user.name ${GITHUB_USER}
  - git remote set-url origin "https://${GITHUB_TOKEN}@github.com/YOUR/PROJECT.git" > /dev/null 2>&1
  - git checkout release
  - echo "//registry.npmjs.org/:_authToken=\${NPM_TOKEN}" >> $HOME/.npmrc 2> /dev/null
deploy:
  provider: script
  script: "yarn run lerna:publish"
  skip_cleanup: true
  on:
    node: "10"
    branch: release
```

Using this config you can deploy (for our purposes, deploy == release) only when a merge happens to a branch called _"release"_. So as you may figure it out, on a workflow like this you will end up having a _parallel_ release branch. This branch will be used to track our releases. 

#### Usage

When you want to trigger a release you should create a PR from _master_ to _release_. When the PR is approved and it's merged this will signal the condition and travis will execute the deploy part. Finally, when the PR got merged you should create another PR from _release_ back to _master_ to get the changelog changes and version bumps.

#### Requirements

Since we are pushing and publising from travis, we will need to pass: a `GITHUB_EMAIL` (this can be deploy@travis-ci.org), `GITHUB_USER` (Deployment Bot), `GITHUB_TOKEN` and finally a `NPM_TOKEN`. 

#### Changelog

Our deploy script is the following: `"yarn run lerna:publish"`. If we take a look to our `package.json` this is what we are running:

`"lerna:publish": "lerna publish prerelease --yes --npm-client yarn --preid beta --npm-tag beta"`

This alone wont do nothing _changelog-related_. Since we are using [chan](https://github.com/geut/chan) we can use one of the npm lifecycle hooks, in this case will be the `postversion` hook and it will look like this:

`"postversion": "chan release --group-changes $(node -p -e \"require('./lerna.json').version\") && git add CHANGELOG.md && git commit -m \"Update: changelog\""`

One thing, we are using postversion and not just the `version` hook due to the missing npm env var `$npm_package_version`. On the other side, it makes sense to get the upcoming version directly from the `lerna.json` if we are using the fixed (default) mode.

> `Chan` works great on monorepos thanks to a new option for releases, called `--group-changes` with in combination of the commit level option `-g package` help us to _separate_ and organize our changes. 

### B - **Half & Half** :star:

The previous strategy has a clear drawback, you can't specify the next semver and it is logical, this is a manual part of the process (at least for the time being :wink:).
But, we have another option. Lerna from version 3 splitted out the version from the publish. In fact they are now 2 different commands. Having this in mind, we could divide the release process in 2 steps that will be quite close each other. One would involve the version bump, push and tag. The other, the npm publish. We can connect both using travis conditions. :cool:

#### Setup

The first part involves versioning, the command (npm script) will look like this:

`"lerna:version": "lerna version prerelease --preid beta"`

Also, it is very convenient to add the following to the lerna.json file to limit the branch where we want to release:

```json
"command": {
  "publish": {
    "allowBranch": "master"
  }
}
```

So far, so good. If we run the `lerna:version` script we would only need to choose/confirm the version bump and we will end up with a new version and a new tag on github.

For the **publish** we will use the new tag on master as a travis condition for deploy:

```yaml
language: node_js
node_js:
  - "10"
cache:
  yarn: true
  directories:
    - "node_modules"
before_install:
  - curl -o- -L https://yarnpkg.com/install.sh | bash -s
  - export PATH="$HOME/.yarn/bin:$PATH"
before_script:
  - yarn run lerna run prepublish
script:
  - yarn test
before_deploy:
  - echo "//registry.npmjs.org/:_authToken=\${NPM_TOKEN}" >> $HOME/.npmrc 2> /dev/null
deploy:
  provider: script
  script: "yarn run lerna:publish"
  skip_cleanup: true
  on:
    node: "10"
    tags: true
```

The relevant parts from the travis setup are the `tags: true` condition and the `NPM_TOKEN` being set as part of the "local" `.npmrc` file. For the conditional, `tags: true` means that travis will only deploy iff the `TRAVIS_TAG` env var is set and the var will only be set if we are building from a git tag. 

Combining the travis condition a github protected branch and the lerna version protection too we can have a quite robust split workflow. 

Finally, let's take a look into the travis deploy script, it will look like this:

`"lerna:publish": "lerna publish from-git --yes --npm-client yarn --npm-tag beta`

Here we are telling lerna to use the version from the current git commit this is a special option very useful for a CI scenario like ours.

#### Usage

Like we said, this is a "2 steps" workflow. 

1. From `master` you can launch the versioning: `yarn run lerna:version`
2. The _automatic_ step, travis will detect a new tag on master and this will trigger a new deploy. :rocket:

#### Requirements

We only need a `NPM_TOKEN` for travis. 

#### Changelog

Last but not least, how do we maintaing a changelog with a strategy like this? Remember, we are using [chan](https://github.com/geut/chan). With this setup, we can't use npm hooks and lerna does not offer any hook (but maybe in the [future](https://github.com/lerna/lerna/issues/643)) so we need to add our own "hooks", we can do something like the following:

```json
{
  "scripts": {
    "lerna:postversion": "chan release --group-changes $(node -p -e \"require('./lerna.json').version\") && git add CHANGELOG.md && git commit -m \"Update: changelog\" && git push --follow-tags",
    "lerna:version": "lerna version prerelease --preid beta && yarn run lerna:postversion"
  }
}
```

...and that's it!

___

_brought to you by:_ 

### GEUT Labs

> The place to try ideas.
