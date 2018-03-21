---
layout: post
title: "Installing with pip without a version"
date: 2017-02-22
---

I ran into this error: "Exception: Versioning for this project requires
either an sdist tarball, or access to an upstream git repository. Are
you sure that git is installed?", and a quick google did not return a
result that really explains what I saw.

# The Error

We received a bug report for Dragonflow, saying that downloading the zip
archive of the project, and then installing it via pip, raised the following
error:

~~~
Processing /tmp/dragonflow-master
    Complete output from command python setup.py egg_info:
    ERROR:root:Error parsing
    Traceback (most recent call last):
      File "/tmp/test/lib/python2.7/site-packages/pbr/core.py", line 111, in pbr
        attrs = util.cfg_to_args(path, dist.script_args)
      File "/tmp/test/lib/python2.7/site-packages/pbr/util.py", line 246, in cfg_to_args
        pbr.hooks.setup_hook(config)
      File "/tmp/test/lib/python2.7/site-packages/pbr/hooks/__init__.py", line 25, in setup_hook
        metadata_config.run()
      File "/tmp/test/lib/python2.7/site-packages/pbr/hooks/base.py", line 27, in run
        self.hook()
      File "/tmp/test/lib/python2.7/site-packages/pbr/hooks/metadata.py", line 26, in hook
        self.config['name'], self.config.get('version', None))
      File "/tmp/test/lib/python2.7/site-packages/pbr/packaging.py", line 725, in get_version
        raise Exception("Versioning for this project requires either an sdist"
    Exception: Versioning for this project requires either an sdist tarball, or access to an upstream git repository. Are you sure that git is installed?
    error in setup command: Error parsing /tmp/pip-2zQwxw-build/setup.cfg: Exception: Versioning for this project requires either an sdist tarball, or access to an upstream git repository. Are you sure that git is installed?
    
    ----------------------------------------
Command "python setup.py egg_info" failed with error code 1 in /tmp/pip-2zQwxw-build/
~~~

When pulling the project with git, the command works. So what's going on?

# Explanation

When you install a package with `pip`, at some point it calls `pbr` and asks
it to guess at the version.

The function code is here, but you don't have to read it. I also removed the
docstring for extra fun.

~~~ python
def get_version(package_name, pre_version=None):
    version = os.environ.get(
        "PBR_VERSION",
        os.environ.get("OSLO_PACKAGE_VERSION", None))
    if version:
        return version
    version = _get_version_from_pkg_metadata(package_name)
    if version:
        return version
    version = _get_version_from_git(pre_version)
    # Handle http://bugs.python.org/issue11638
    # version will either be an empty unicode string or a valid
    # unicode version string, but either way it's unicode and needs to
    # be encoded.
    if sys.version_info[0] == 2:
        version = version.encode('utf-8')
    if version:
        return version
    raise Exception("Versioning for this project requires either an sdist"
                    " tarball, or access to an upstream git repository."
                    " Are you sure that git is installed?")
~~~

What this function does is (and this is all documented):
#. Try reading it from the environment
#. Try reading it from the metadata
#. Try reading it from git
#. Fail miserably with a nice error

## Try reading it from the environment

This one is fairly straightforwards. If `PBR_VERSION` exists, use that. Otherwise,
if `OSLO_PACKAGE_VERSION` exists, use that. Otherwise, return `None` so we know
to keep trying. If you found something, return it.

## Try reading it from the metadata

I won't go into the implementation of this method. Again, it is well documented,
and called `_get_version_from_pkg_metadata`.

The gist of it is this: Try reading the info from the file `PKG-INFO`. If that
file doesn't exist or does not parse properly, try reading the info from the
file `METADATA`.

In the parsed info, verify that the package name in the file is the package
we're looking at (in this case, 'dragonflow'). If it is, read the `version`
field. If the file is empty, doesn't have a version field, or belongs to a
different package, return `None`. We'll fail miserably later.

## Try reading it from git

This one is complete black magic.

This heuristic assumes that the git branch is tagged somewhere along the way.
In our bug's case, we don't have a git tag. We don't have a git. We downloaded
a zip archive. It doesn't contain the git data.

First check if this commit is tagged. If it is, take that tag. This is done
with the command `git describe --exact-match`, and expecting it to raise an
error if there is no tag.

Otherwise, count up to the last commit on the branch that is tagged. Take that
tag (call it... `<tag>`), and the distance (call it... `<distance>`. I am not very
good at naming.), and construct a new version from it: `<tag>-dev<distance>`.

# Quick Fix

Clone the git repo! Instead of downloading the zip file, run `git clone`, and
then everything should work as expected.

Now, yes. You can override it with the environment variables above. Don't do
that! Cloning the git repo and having a real versioning means that when you
move ahead in the branch, the version is updated, and you have a general idea
of where you are, and what's installed.

So again, clone the git repo!

If you need help with git, I will recommend this amazing book that taught me
git in a week: [progit](https://git-scm.com/book/en/v2).
