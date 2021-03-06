# The Gradle Git Repo Plugin

This plugin allows you to add a git repository as a maven repo, even if the git
repository is private, similar to how CocoaPods works.

Using a github repo as a maven repo is a quick and easy way to host maven jars.
Private maven repos however, aren't easily accessible via the standard maven
http interface, or at least I haven't figured out how to get the authentication
right. This plugin simply clones the repo behind the scenes and uses it as a
local repo, so if you have permissions to clone the repo you can access it.

This plugin lets you tie access to your repository to github accounts, or any git repository
seamlessly. This is most useful if you've already set up to manage distribution
this way. Deliver CocoaPods and Maven artifacts with the same system, then sit
back and relax.

## Building

Run `gradle build` to build, and `gradle publish` to publish. Sadly, this plugin no longer
uses itself to publish itself :(.

## Usage
[Gradle Plugin - com.github.unafraid.gradle.git-repo-plugin](https://plugins.gradle.org/plugin/com.github.unafraid.gradle.git-repo-plugin)

This plugin needs to be added via the standard plugin mechanism with this buildscript in your top level project
# Build script snippet for use in all Gradle versions
	buildscript {
		repositories {
			maven {
				url "https://plugins.gradle.org/m2/"
			}
		}
		dependencies {
			classpath "gradle.plugin.com.github.unafraid.gradle:gradle-git-repo-plugin:2.0.4"
		}
	}

# Build script snippet for new, incubating, plugin mechanism introduced in Gradle 2.1:
	plugins {
		id "com.github.unafraid.gradle.git-repo-plugin" version "2.0.4"
	}

and then apply the plugin

	apply plugin: "com.github.unafraid.gradle.git-repo-plugin"


### Depending on git repos

This plug adds `github` and `git` methods to your repositories block

	repositories {
		github("layerhq", "maven-private", "origin", "master", "releases")
		git("https://some/clone/url.git", "arbitrary.unique.name", "origin", "master", "releases")
	}

Add either alongside other repositories and you're good to go. The `github` variant is
just a special case of `git`, they both do the same thing.

### Publishing to github repos

Publishing is a bit less seamless, mostly because there isn't one single way to
handle publishing in gradle (also the maven-publish plugin is infuratingly
tamper-proof). You're expected to have a task called `publish` by default, that
publishes to the locally cloned repo. That task gets wrapped into a
`publishToGithub` task that handles committing and pushing the change. First, add this
configuration block, which will use github by default:

	gitPublishConfig{
		org = "layerhq"
		repo = "maven-private"
		upstream = "origin"
		branch = "master"
	}

The `maven-publish` plugin defines a publish task for you, so you just need to
supply the right url in the publishing block

	task sourceJar(type: Jar) {
		classifier "sources"
		from sourceSets.main.allJava
	}
	 
	publishing {
		publications {
			mavenJava(MavenPublication) {
				from components.java

				artifact sourceJar
			}
		}
		repositories {
			maven {
				url "file://${gitPublishConfig.home}/${gitPublishConfig.org}/${gitPublishConfig.repo}/releases"
			}
		}
	}

Then you can run

	gradle publishToGithub

You can also run 

	gradle publish

to stage a release in the local github repo and commit it manually.


A version of this with the `maven` plugin might look like

	String url() {
		String org = gitPublishConfig.org
		String repo = gitPublishConfig.repo
		String repoHome = gitPublishConfig.home
		return "file://$repoHome/$org/$repo/releases"
	}
	
	task publishJar(type: Upload, description: "Upload android Jar library") {
		configuration = configurations.sdkJar
		repositories {
			mavenDeployer {
				repository(url: url())
			}
		}
	}

## Settings

The following gradle properties affect cloning dependencies

- **offline** when defined, no network operations will be performed, the repos will be assumed to be in place
- **home** the base directory for cloning git repos, ~/.gitRepos by default


For publishing, the following configuration is supported, to allow non-github repos and other settings

	gitRepoConfig {
		// Mandatory
		org = "myorg"
		repo = "myrepo"

		// Optional
		gitUrl = "" //used to replace git@${provider}:${org}/${repo}.git
		provider = "github.com" // or "gitlab.com", or any other github like
		upstream = "origin"
		branch = "master"
		home = "${System.properties['user.home']}/.gitRepos" base directory for cloning
		publishAndPushTask = "publishToGithub" // the name for the full publish action
		publishTask = "publish" //default publish tasks added by maven-publish plugin
		offline = false // if true, no git clones will be performed, the repo will be assumed to be there
		useCache = true // If false, no caching is used when used in multiple projects may cause some delay
	}

## Futures

It would be nice to make publishing seamless and completely
hide the locally cloned repo. That might require reimplementing maven
publishing though. The `maven-publish` plugin isn't amenable to having its
settings messed with after it's been applied unfortunately.

After long-term use, your git repo can get very large, and cloning it becomes slow

## Credits

Douglas Rapp

- http://github.com/drapp
- http://twitter.com/platykurtic
- douglas.rapp@gmail.com

## License

The gradle git repo plugin is available under the Apache 2 License. See the LICENSE file for more info.
