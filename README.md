# git-refrepo-scripts
Scripts to manage a monolithic or nested-fanout reference repository to speed
up Git clones and fetches.

It aims to align with related evolution of Jenkins Git Client plugin proposed in
[jenkinsci/git-client-plugin#644](https://github.com/jenkinsci/git-client-plugin/pull/644)

* The `register-git-cache.sh` script is not (yet?) a git method.
  This script is here to allow managing a git repository in the directory it
  resides in as a sort of cache, usable as a reference git repo for faster
  clones e.g. in a CI environment.

    * Thanks to bits of wisdom at https://stackoverflow.com/a/57133963/4715872
    and https://pcarleton.com/2016/11/10/ssh-proxy/ I now know that if we have
    overloaded our firewall banging SSH repos at `github.com:22` we can also
    fetch over make-believe HTTPS port including over a proxy like Squid with
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
