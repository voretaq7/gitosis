`gitosis` -- software for hosting `git` repositories
====================================================

Manage `git` repositories, provide access to them over SSH, with tight access
control and not needing shell accounts.

> **Note**
>
> Documentation is still lacking, and non-default configurations (e.g. config
file, repositories, installing in a location that is not in `PATH`) basically
have not been tested at all. Basic usage should be very reliable -- the
project has been hosting itself for a long time. Any help is welcome.

`gitosis` aims to make hosting `git` repos easier and safer. It manages
multiple repositories under one user account, using SSH keys to identify
users.  
End users do not need shell accounts on the server, they will talk to one
shared account that will not let them run arbitrary commands.

`gitosis` is licensed under the GPL, see the file `COPYING` for more
information.

You can get `gitosis` via `git` by saying:

    git clone https://github.com/voretaq7/gitosis.git

And install it via:

    python setup.py install

Though you may want to use e.g. `--prefix=`.

Setting up
----------

First, we will create the user that will own the repositories. This is
usually named `git`, but any name will work, and you can have more than one
per system if you really want to. The user does not need a password, but
does need a valid shell (otherwise, SSH will refuse to work).  
Create a dedicated service account. **DO NOT** use a normal user account.

The git repositories will be stored in the service account user's home
directory, so a location such as `/srv/example.com/git` or `/var/git`
is recommended. Create the user with :

    sudo adduser \
        --system \
        --shell /bin/sh \
        --gecos 'git version control' \
        --group \
        --disabled-password \
        --home /srv/example.com/git \
        git

or equivalent commands on your system.

Initial Configuration
----------

You will need an SSH public key to continue. If you don't have one,
you need to generate one. See the man page for `ssh-keygen`, and
you may also be interested in `ssh-agent`.  
Create the SSH key on your personal computer and protect the
*private* key well -- that includes not transferring it over the
network.

Next, we need to set things up for the newly-created `git` user.  
The following command will create a `~/repositories` that will hold
the `git` repositories, a `~/.gitosis.conf` that will be a symlink
to the actual configuration file, and it will add the SSH public
key to `~/.ssh/authorized_keys` with a `command=` option that
restricts it to running `gitosis-serve`.  
Run:

    sudo -H -u git gitosis-init <FILENAME.pub
    # (or just copy-paste the public key when prompted)

You can now `git clone git@SERVER:gitosis-admin.git`, and you get
a repository with SSH keys as `keys/USER.pub` and a `gitosis.conf`
where you can configure who has access to what.

> **warning**
>
> For now, `gitosis` uses the `HOME` environment variable to
locate where to write its files. If you use `sudo -u` without
`-H`, `sudo` will leave the old value of `HOME` in place, and
this will cause trouble. There will be a workaround for that later
on, but for now, always remember to use `-H` if you're sudoing
to the account.

You should always edit the configuration file via `git`.  
The file symlinked to `~/.gitosis.conf` on the server will be
overwritten when pushing changes to the `gitosis-admin.git`
repository.

Edit the settings as you wish, commit and push. That's pretty much it!
Once you push, `gitosis` will immediately make your changes take
effect on the server.



Security Recommendations
----------
The gitosis system and managed repositories will be available using
your system's regular SSH daemon. This may be what you want, however
if you don't expose SSH publicly you may wish to set up a dedicated
SSH daemon exclusively for gitosis.  
For best security this should run on an alternate port, only allow the
`git` user to log in, and not allow the use of interactive passwords.

Some recommended `sshd_config` settings:  

    AllowUsers git
    AllowAgentForwarding no
    AllowTcpForwarding no
    X11Fordwarding no
    PermitRootLogin no
    PasswordAuthentication no
    ChallengeResponseAuthentication no





Managing it
-----------

To add new users:

-   add a `keys/USER.pub` file
-   authorize them to read/write repositories as needed  
    (or just authorize the group `@all`)

To create new repositories, just authorize writing to them
and push. It's that simple!  
For example: let's assume your username is `jdoe` and you want
to create a repository `myproject`. In your clone of
`gitosis-admin`, edit `gitosis.conf` and add:

    [group myteam]
    members = jdoe
    writable = myproject

Commit that change and push, then create the initial commit
in your local repository and push it:

    mkdir myproject
    cd mypyroject
    git init
    git remote add myserver git@MYSERVER:myproject.git
    # do some work, git add and commit files
    git push myserver master:refs/heads/master

That's it. If you now add others to `members`, they can write
to that repository too.

Optional Configuration
---------------------

Using `git daemon`
------------------

Anonymous read-only access to `git` repositories is provided by
`git daemon`, which is distributed as part of `git`. But `gitosis`
will still help you manage it: setting `daemon = yes` in your
`gitosis.conf`, either globally in `[gitosis]` or per-repository
under `[repo REPOSITORYNAME]`, makes `gitosis` create the
`git-daemon-export-ok` files in those repository, thus telling
`git daemon` that publishing those repositories is ok.

To actually run `git daemon` refer to your operating system's
instructions for running daemons at startup.

Note that this short snippet is not a substitute for reading
and understanding the relevant documentation.

Using gitweb
------------

`gitweb` is a CGI script that lets one browse `git` repositories
on the web. It is most commonly used anonymously, but you could
also require authentication in your web server, before letting
people use it.

`gitosis` can help here by generating a list of projects that
are publicly visible. Simply add a section
`[repo REPOSITORYNAME]` to your `gitosis.conf`, and allow
publishing with `gitweb = yes` (or globally under `[gitosis]`). 
You should also set `description` and `owner` for each repository.
