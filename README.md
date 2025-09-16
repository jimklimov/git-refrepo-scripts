# git-refrepo-scripts
Scripts to manage a monolithic or nested-fanout reference repository to speed
up Git clones and fetches.

It aims to align with related evolution of Jenkins Git Client plugin proposed in
[PR jenkinsci/git-client-plugin#644](https://github.com/jenkinsci/git-client-plugin/pull/644)
to solve formal [issue JENKINS-64383](https://issues.jenkins.io/browse/JENKINS-64383).

## Notes about Git Reference Repositories (refrepo's)

For background, see e.g.:

* https://support.cloudbees.com/hc/en-us/articles/115001728812-Using-a-Git-reference-repository
* https://randyfay.com/content/reference-cache-repositories-speed-clones-git-clone-reference
* https://randyfay.com/content/git-clone-reference-considered-harmful   for some caveats

In particular, note that usage of reference repositories does its magic by
having a newly cloned (and later updated?) repository refer to bits of code
present in some filesystem path, rather than make a copy from the original
remote repository again - so saving network, disk space and maybe time.
Corollaries:

* the reference repo should be in the filesystem (maybe over NFS) and
  at the same locally resolved FS path if shared across build agents
* note that `git` parsing of the reference repo involves a lot of I/O
  to the index data, which is a performance constraint specifically
  with NFS-shared locations (those filesystems are usually not cached in
  RAM by clients), and is the reason why we want to avoid known-useless
  operations and use nested directories dedicated just to closely related
  remote URLs - different paths to same project, or its forks that share
  most content
* the reference repo should be at least readable to the build account
  when used in Jenkins
* files (and contents inside) should not disappear or be renamed over
  time (garbage collection, pruning, etc. do that) or the cloned repo
  will become invalid (not a big issue for workspaces that made a build
  in the past and are not reused as such, but may be a problem to remake
  the same run without extra rituals to create a coherent checkout;
  not sure if this is also a problem for reusing the workspace for later
  runs of a job, with same or other commits)
    * There is a "git disassociate" command for making a workspace standalone
      again, by copying into it the data from a reference repo - forfeiting
      the disk savings, but keeping the network/time improvements probably;
      this is not integrated into Jenkins Git client side, AFAIK
* the advanced option to use a reference repo is only applied during
  cloning - existing workspaces must be remade to try it out

## Notable scripts offered by this repository

* The `register-git-cache.sh` script is not (yet?) a git method.
  This script is here to allow managing a git repository in the directory it
  resides in as a sort of cache, usable as a reference git repo for faster
  clones e.g. in a CI environment. It was originally proposed for improving
  performance in Jenkins jobs using a single configuration (e.g. generated
  automatically from OrgFolders and so inheriting the Git configuration
  specified there).
  For more details, see comments and code in the script; some are summarized
  in this README.

    * After you register the repositories you want to track, you can call
      this script from a crontab or a Jenkins job (see below) to occasionally
      update the cache.
    * If you host this on a ZFS-capable system (even if remotely accessed like
      over NFS), a dedicated dataset for the cache is recommended - this script
      will then snapshot it after updates, to save a consistent filesystem state
      every time.
    * Thanks to bits of wisdom at https://stackoverflow.com/a/57133963/4715872
      and https://pcarleton.com/2016/11/10/ssh-proxy/ I now know that if we have
      overloaded our firewall banging SSH repos at `github.com:22` we can also
      fetch over make-believe HTTPS port including over a proxy like Squid (with
      `ProxyCommand` line un-commented below):
      ````
      ### ~/.ssh/config
      Host github.com
          Hostname ssh.github.com
          #ProxyCommand nc -X connect -x <PROXY-HOST>:<PROXY-PORT> %h %p
          Port 443
          ServerAliveInterval 20
          User git
          IdentitiesOnly yes
          IdentityFile   ~/.ssh/id_rsa_jenkins
      ````
    * The script run-time is largely configured by environment variables,
      which are easier to integrate with Jenkins `sh` and `withEnv` steps,
      detailed in header comments of the script.
    * This project was tested to work on Windows deployments; and is regularly
      used on Linux and OpenIndiana (illumos/Solaris) Jenkins controllers.

* The `git-clone-rr` is a git method (can be used as `git clone-rr` if
  placed into `PATH`) that can use a `GIT_REFERENCE_REPO_DIR` envvar or
  intercepts the `--reference(-if-able) <dir>` argument with parameterized
  format as supported by Jenkins git-client-plugin (after PR #664) to use
  a monolithic or nested reference repository directory tree maintained by
  `register-git-cache.sh` using the same string for `<dir>` configuration.
  This helps in mass-cloning operations in shell scripts.

* The `Jenkinsfile-rescan-MBPs` provides a sample pipeline script job that
  can discover previously not tracked SCM URLs in recent builds on Jenkins,
  and call the script above deployed on a shared NAS to provide the Git
  reference repo to the whole CI farm. As another goal, this job allows
  to regularly poll organizations on SCM platforms to discover new repos,
  and new branches in the generated MBP (Multi-Branch-Pipeline) jobs --
  something that Jenkins did very rarely (once a day) relying on webhooks.
  Alas, those don't exist for tightly firewalled CI farms.

* It is possible to also define a simple job on your Jenkins instance,
  utilizing an agent that has write-access to the persistent git-cache
  location, to maintain this repository (trigger rescans so the cache
  stays relevant). Be sure to set `REFREPODIR_MODE` for those runs, either
  from environment or by using a `.gitcache.conf` file in the refrepo dir.
  The `Jenkinsfile-update-gitcache` offers a starting point for such job.

* With 2.36.x and newer Git versions, if your reference repository
  maintenance script runs as a different user account than the Jenkins server
  (or Jenkins agent) on the same system, safety checks about `safe.directory`
  from 2.35.2 (see https://github.blog/2022-04-18-highlights-from-git-2-36/ and
  https://git-scm.com/docs/git-config/#Documentation/git-config.txt-safedirectory)
  can be disabled by configuring each such user account:
  ````
  :; git config --global --add safe.directory '*'
  ````

Hope this helps,
Jim Klimov
