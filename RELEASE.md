# Releasing Habitat

This document contains step-by-step details for how to release Habitat. All components are released
from the master branch on a bi-weekly schedule occurring every other Monday.

## Releasing Launcher

The [`core/hab-launcher` package](https://bldr.habitat.sh/#/pkgs/core/hab-launcher), which contains
the `hab-launch` binary has a separate release process. See
[its README](components/launcher/README.md) for details. The Buildkite pipeline does not build a `core/hab-launcher` package; that must currently be built out-of-band and uploaded to Builder. The Buildkite pipeline _will_ however take care of _promoting_ it for you. The pipeline will guide you as appropriate.

## Compiling Release Notes

Start with the [autogenerated CHANGELOG](https://github.com/habitat-sh/habitat/blob/master/CHANGELOG.md) - Copy and paste this into a google doc for collaboration.

Look through the merged pull requests and see if any of them is marked with the [C-breaking] label - if they are, make sure to mark their entry in the release notes with [BREAKING CHANGE].

If you happen to be aware of features or bugs and feel there are user impacting words that need to be expressed, this is your opportunity to express them.

Ask all team members to review the google doc and

1. Add in any details to the changelog for any changes they made
2. Add a sentence or two explaining how the change was tested/validated

## Submit a Release Notes Blog Post PR

If you haven't posted to the blog before, see [this article](https://github.com/habitat-sh/habitat/tree/master/www#how-to-add-a-blog-post) since there may be some setup required. Also, don't forget to build and test your post locally before merging it.

If one has not already been opened, create a "Release Notes" blog post PR. Add a file to the Habitat repo: `www/source/blog/2018-02-20-0540Release.html.md`. Of course you will change the date and version to match the reality that is manifesting itself in your present moment. This post should begin with the following structure:

```
---
title: Habitat <VERSION> Released
date: <yyyy-mm-dd>
author: Tasha Drew
tags: release notes
category: product
classes: body-article
---

Habitat <VERSION> Release Notes

We are happy to announce the release of Habitat <VERSION>. We have a number of new features as well as bug fixes, so please read on for all the details. If you just want the binaries, head on over to [Install Habitat](https://www.habitat.sh/docs/using-habitat/#install-habitat). Thanks again for using Habitat!

Highlights:
* [BREAKING CHANGE] SOME BREAKING CHANGE
```

**Important:** Filter all closed PRs labeled `C-breaking` and ensure any and all of these PRs is mentioned as a `[BREAKING CHANGE]` in the post highlights.

## Prepare Master Branch for Release

1. Call for a "Freeze" on all merges to master (announce in the internal Chef slack team as well as in the Habitat slack team)

1. Clone the Habitat repository if you do not already have it

    ```
    $ git clone git@github.com:habitat-sh/habitat.git ~/habitat
    ```

1. Ensure you are on the master branch and have the latest of `~/habitat`

    ```
    $ cd ~/habitat
    $ git checkout master
    $ git pull origin master
    ```

1. Create a new release branch in the Habitat repo. You can call this branch whatever you wish.

    ```
    $ cd ~/habitat
    $ git checkout -b <branch>
    ```

1. Remove the `-dev` suffix from the version number found in the `VERSION` file. *Note*: there must not be a space after the `-i`.

    ```
    $ sed -i'' -e 's/-dev//' VERSION
    ```

1. Commit `VERSION` changes and push your branch
1. Issue a new PR with the `Expeditor: Exclude from Changelog` label and await approval (in the form of a [dank gif](http://imgur.com/X0sNq)) from two maintainers
1. Pull master once again once the PR is merged into master
1. Create & push a Git tag

    ```
    $ make tag-release
    $ git push origin --tags
    ```

Once the release tag is pushed, Buildkite and AppVeyor builds will be triggered on the release tag. AppVeyor builds are currently very prone to timing out,
so set a 1-hour timer to go and check on them. If they do time out, you just have to restart them and hope. You may also want to set up [email notifications](https://ci.appveyor.com/notifications).

You can view/adminster Buildkite builds [here](https://buildkite.com/chef/habitat-sh-habitat-master-release).

The release tag builds will upload all release binaries to a channel named `rc-[VERSION]` and the `hab` cli will be uploaded but _not_ published to the `stable` Bintray repository. These builds can take about 45 minutes to fully complete. Keep an eye on them so we can validate the binaries when they finish.

## Validate the Release

For each platform ([darwin](https://bintray.com/habitat/stable/hab-x86_64-darwin), [linux](https://bintray.com/habitat/stable/hab-x86_64-linux), [windows](https://bintray.com/habitat/stable/hab-x86_64-windows)), download the latest stable cli version from [Bintray](https://bintray.com/habitat/stable) (you will need to be signed into Bintray and a member of the "Habitat" organization). These can be downloaded from the version files page but are unpublished so that our download page does not yet include them. There may be special behavior related to this release that you will want to validate but at the very least, run `hab studio enter` and make sure:

Due to historical contingencies, evaluating the release is currently a bit tricky.

You need to set `HAB_INTERNAL_BLDR_CHANNEL` and `CI_OVERRIDE_CHANNEL` to the name of the release channel (you _may_ also need to set `HAB_STUDIO_SECRET_HAB_INTERNAL_BLDR_CHANNEL` and `HAB_STUDIO_SECRET_CI_OVERRIDE_CHANNEL`). If a new Launcher is in the release channel, you should be fine; however, since that should be rare, you may have some additional work.

NOTE: If you are running `sudo hab studio enter` with all the required environmental variables set, but it's still telling you that it cannot find the package in stable, try `sudo -E hab studio enter`.

In a previous release, we were able to validate things on Linux by re-using a chroot studio and installing a Launcher out-of-band. You can probably create a new studio, enter it with `HAB_STUDIO_SUP=false`, manually install the latest stable Launcher (if a new one isn't part of the current release), exit the studio, then re-enter with `HAB_STUDIO_SUP` unset (but with all the override variables mentioned above set). This should reuse the Launcher you just installed, but pull in additional artifacts as needed from your release channel.

See https://github.com/habitat-sh/habitat/issues/4656 for further context and ideas.

Then you can actually exercise the software as follows:

1. It pulls down the correct studio image
1. That studio's `hab` is at the correct version
1. A `sup-log` shows a running supervisor (if `sup-log` does not show a supervisor running, run `hab install core/hab-sup --channel release_channel` then `hab sup run`)
1. Verify that the supervisor is the correct version by running `ls /hab/pkgs/core/hab-sup`

When testing the linux studio, you will need to `export CI_OVERRIDE_CHANNEL` to the rc channel of the release. So if you are releasing 0.75.2, the channel would be `rc-0.75.2`.

### Addressing issues with a Release

If you find issues when validating the release binaries that must be fixed before promoting the release, you will need to fix those issues and then have Travis and Appveyor rerun the deployment. After you merge the necessary PRs to fix the release issues:

1. Delete the local release tag: `git tag -d <tag>`
2. Delete the remote tag: `git push origin :<tag>`
3. Tag the release again: `make tag-release`
4. Push that tag: git push origin --tags

# Post-Release Tasks
The Buildkite release is fairly-well automated at this point, but once it is complete, there are still a few remaining manual tasks to perform. In time, these will be automated as well.

## Update Builder Bootstrap Bundle

Once the travis linux deployment has completed, we generate a release bundle of all Habitat and Builder components which are uploaded to an S3 bucket which we read from when we bootstrap new nodes. This bundle is useful if you are bootstrapping in an environment which doesn't have access to Builder or there simply isn't a Builder instance in existence (ah, those were the days).

NOTE: Do this step from either a Linux VM or in a studio.

1. Configure your AWS credentials in your environment
1. Execute the script that currently lives in the [builder](https://github.com/habitat-sh/builder) repository:

    ```
    $ cd /path/to/builder-repo
    $ sudo AWS_ACCESS_KEY_ID=<...> AWS_SECRET_ACCESS_KEY=<...> terraform/scripts/create_bootstrap_bundle.sh <HABITAT_VERSION>'
    ```

## Update Homebrew Tap

We have our own [Homebrew tap](https://github.com/habitat-sh/homebrew-habitat) that will need
updating. You will need the following bits of information for the latest stable macOS Bintray artifact:

* the new version number
* the new release
* the SHA256 of the Bintray zip file

With those in hand, update the
[formula](https://github.com/habitat-sh/homebrew-habitat/blob/master/Formula/hab.rb#L3-L5),
and merge the changes to the master branch.

(This will be a temporary state of affairs; I'll be talking with Engineering Services soon to get their help with automating this, as well as other parts of our release process.)

## Rerun Chocolatey Validation Tests

The Appveyor release will automaticaly publish to Chocolatey. However because we do not publish to bintray right away, Chocolatey's package validation will fail when it performs a test install.

After we perform the final `make publish-release` above, we need to tell Chocolatey to rerun its validation so that they can pass and be visible to the public. To do this:

1. Go to `https://chocolatey.org/`
1. Login with the `habitat` account (creds can be found in 1password).
1. Go to `https://chocolatey.org/account/Packages`
1. Click on the `Habitat` package
1. Scroll to the bottom of te package page and check `rerun tests` and submit.

The tests should now pass and the package should become publicly visible.

## Publish Release

Create release in [GitHub](https://github.com/habitat-sh/habitat/releases)

On the Github releases page, there should already be a tag for the release (pushed up previously).
Draft a new Release, specify the tag, and title it with the same (eg, 0.18.0). Then hit Publish Release.

## Update the Acceptance environment with the new hab-backline

After the new hab is released and there is a new hab-backline in stable, the acceptance environment will also need to be updated. In order to do this, the simplest steps (from a Linux machine) are:

```
sudo hab pkg install core/hab-backline/<latest version>
cd /hab/cache/artifacts
hab pkg upload -u https://bldr.acceptance.habitat.sh -c stable -z <AUTH_TOKEN> core-hab-backline-<latest version>-x86_64-linux.hart

```

*Important*: Don't forget to include the `-c stable` in the upload step above!

## Update the Docs

Assuming you've got a locally installed version of the `hab` CLI you just released, you can update the CLI documentation in a separate PR. To do that:

```
cd www
make cli_docs
make template_reference
```

Verify the diff looks reasonable and matches the newly released version, then submit your PR.

# Drink. It. In.

## Bump Version

1. Update the version number found in the `VERSION` file to the next target release and append the `-dev` suffix to that number
1. Issue a PR and merge it yourself

> Example: If the release version was `0.9.0` then the contents of `VERSION` might read `0.10.0-dev` if your next target is `0.10.0`.

## Update the Acceptance environment with the new hab-backline

Now there's yet another new version of hab-backline, in unstable. So off to the races again for acceptance. This time you'll install the hab backline from _unstable_ channel.  The other steps are the same:

```
sudo hab pkg install -c unstable core/hab-backline
cd /hab/cache/artifacts
hab pkg upload -u https://bldr.acceptance.habitat.sh -c stable -z <AUTH_TOKEN> core-hab-backline-<latest version>-x86_64-linux.hart

```

*Important*: Don't forget to include the `-c stable` in the upload step above!

## Promote the Builder Worker

Now that the release is stable, it is time to promote the workers.

In the Builder Web UI, go to the builds page for `habitat/builder-worker` and check that the latest version is a successful recent build.

Promote the release. Wait for a few minutes, then perform a test build. Check the build log for the test build to confirm that the version of the Habitat client being used is the desired version.

# If your Release is going to cause downtime

1. Put a scheduled maintanence window into PagerDuty so the on-call doesn't go off.
1. Pre-announce outage on Twitter & Slack #general channel. There is no hard rule for a length of time you need to do this ahead of a release outage.
1. When your release begins, post announcement in our statuspage.io with outage information.
1. Per [ON_CALL.md](#ON_CALL.md) you are responsible for updating status changes in statuspage.io as the downtime proceeds. You are not responsible for regular minutes or responding in any other venue.
1. When the downtime is over, announce the end of the outage via statuspage.io. It will automatically post an announcement to #general and twitter.

# Release Notification

1. Create a new post in [Habitat Announcements](https://discourse.chef.io/c/habitat)
1. Link forum post to the github release
1. Link github release to forum post
1. Tweet a link to the announcement @habitatsh
1. Make sure the blog post PR with the release announcements is merged and published.
1. Announce that the "Freeze" on merges to master is lifted in both the Chef internal slack team and in the Habitat slack team.

# Update Cargo.lock

1. In the [habitat](https://github.com/habitat-sh/habitat) repo, run `cargo update`, `make clean`, `make` and `cargo build -p habitat-launcher`.
1. If there are warnings or errors that are simple, fix them. Otherwise, lock the appropriate versions in `Cargo.toml` files that lets the build succeed and file an issue to resolve the failure and relax the version lock.
1. Open a PR for the `Cargo.lock` updates and any accompanying fixes which are necessary.
1. Repeat with the [core](https://github.com/habitat-sh/core) and [builder](https://github.com/habitat-sh/builder) repos (omit the `habitat-launcher` build).
