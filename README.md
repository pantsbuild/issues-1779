This repo serves as a minimal example of configuring pants and ivy to use local jars for stability and predictability of resolves.

The main idea here is to 1st look for jars in the local git repo and only fall back to maven central (or your own nexus or artifactory) when a jar is not found.  In the steady state, jars will be found locally.  Only when a new dep is added to a BUILD or an existing dep's version is changed, will remote maven repos be consulted.  When this is done, if the resolve worked and the resulting jars / classpaths are deemed acceptable, the in-repo ivy cache can be checked in.  Further resolves will now use the checked-in jars.

A few things to try out to sample the workflow:

1. Examine the resolve of already checked-in jars:

   ```console
   $ git clean -fdx
   $ find build-support/ivy/cache/ > /tmp/before
   $ ./pants resolve :xalan
   $ git status
   $ find build-support/ivy/cache/ > /tmp/after
   $ diff /tmp/before /tmp/after
   ```

   You'll notice a few things here.  First, the git status command will show no repo changes after the resolve.  However the diff of the `build-support/ivy/cache/` contents will show new files after the resolve.  These files are `.gitignore`d since they contain machine-specific paths and volatile timestamps.  As it turns out, ivy generates these files very quickly when not present, so this has a minimal impact on 1st-resolve times.

2. Run a resolve that finds new jars:

   ```console
   $ cat << EOF >> BUILD.tools
   jar_library(
     name='jar-tool',
     jars=[
       jar('org.pantsbuild', 'jar-tool', '0.0.6')
     ]
   )
   EOF
   $ ./pants resolve :jar-tool
   ```

   This resolve will take a bit longer than the previous since jar-tool and its deps have not been locally cached.  You can see the new ivy cache content with `git status` and check-it in with `git add BUILD.tools build-support/ && git commit -m 'Add the :jar-tool dep.'`

NB: This example uses no special tricks, jars are cached in the `build-support/ivy/cache/` dir.  This might be fine for small projects (not many jar deps and/or the jar dep versions don't change much over time), but can lead to well known bloat of repos and bloat in clone times for repos with alot of history of changing jar binaries.  There are a few ways to keep the storage of the cache out of the git repository:

1. Make `build-support/ivy/cache/` a symlink to a path outside the git repo.
  
   This can work well given you have some rules / scripts / infra around how your repo should be checked out.  You might dictate (or encode in a setup script), that your git code repo should be cloned side-by-side with an svn repo holding the ivy cache dir.
   Any new poms/jars that result from a `build-support/ivy/cache/` miss and resolve via maven central would then need to be svn comitted and some way to ensure timely updates of the svn repo put in place (a cron/anacron job might be one way to freshen the svn repo regularly).

2. Use [git-annex](https://git-annex.branchable.com/) and a special remote to store the `build-support/ivy/cache/` content.

   Special remotes are listed [here](https://git-annex.branchable.com/special_remotes/), but obvious choices include directory, rsync, bittorrent and S3.
