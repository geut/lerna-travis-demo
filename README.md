# Playing with chan on monorepos

> :warning: This is a demo project showcasing: lerna + chan + travis + npm

Okay, let's say you have a monorepo and you are happy with it. Everything seems to be OK but one day your monorepo open source project starts to be used by someone else so you say let's automatize things. 

Probably at this point, some steps are _manually_ controlled, like `publish` for example. At the same time, you are a cool person and make use of travis for CI. 

So you say to yourself, what about _CDing_? Can I release a new _lerna-managed_ package version from Travis? Is that even possible?

Ok, lets figure it out. Oh! But one more thing. I've mentioned that you are cool person, and that means your project will maintain a `CHANGELOG.md` file. OMG, that is so cool. Luckily for us, the GEUT team is here to help us, they have built this little tool called [chan](https://github.com/geut/chan) which can be used to easily create and **maintain** a changelog file based on [keepachangelog](https://keepachangelog.com/en/1.0.0/) directives. Which in the end, results in a human readable list of changes about the progress of our project.

## How?

TBD
- step 1
- step 2

___

_brought to you by:_ 

### GEUT Labs

> The place to try ideas.
