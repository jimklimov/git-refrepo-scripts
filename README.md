# git-refrepo-scripts
Scripts to manage a monolithic or nested-fanout reference repository to speed up Git clones and fetches

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

Hope this helps,
Jim Klimov
