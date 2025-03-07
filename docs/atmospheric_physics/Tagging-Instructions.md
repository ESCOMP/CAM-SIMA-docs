# How to properly tag an atmospheric_physics commit

This page lists instructions for how to create a new atmospheric_physics tag.  A few rules:

1.  Tags should always be annotated (no lightweight tags).
2.  Tags should follow the naming convention listed [below](#tag-naming-conventions).
3.  All merge commits into the main branch should be tagged.
4.  Tags can only be created by people who have write access.

## How to create a git tag via the command line

You can create a new atmospheric_physics tag, assuming you have git installed on your local machine, by doing the following:

**1.  Download the latest version of the repo:**

```
git clone https://github.com/ESCOMP/atmospheric_physics.git
cd atmospheric_physics
```

If you are tagging a development commit, then also make sure to checkout the development branch:

```
git checkout development
```

**2.  Find the commit hash you want to tag.  This can be done by opening the git log like so:**

`git log`

And finding the commit hash you are tagging, along with the commit message.  If you simply want to tag the head of the repo (i.e. the latest commit), then do:

`git log --oneline -1`

And use the provided hash and message.

**3.  Create the tag:**

`git tag -a <tag> <commit_hash> -m '<commit_message>'`

Where `<tag>` is the new tag name (which should follow the naming convention shown below), `commit_hash` is the commit hash you found in step 2, and `commit_message` is the message/description associated with that commit.

**4.  Push the new tag back to the repo:**

`git push origin <tag>`

After which the new tag should now exist in the ESCOMP/atmospheric_physics repo.

## Tag naming conventions

All ESCOMP/atmospheric_physics tags for the main branch should look like the following:

`atmos_physX_YY_ZZZ`

While all tags for the development branch should be:

`dev_atmos_physX_YY_ZZZ`

Where `X` is the major release version (which should almost always be left as-is), `YY` is for whenever a new physics scheme or suite is added, and `zz` is for any other minor release/bug fix.

So, for example, if the latest tag is:

`atmos_phys0_05_025`

And you want to tag a new commit that fixed a bug, then the new tag should be:

`atmos_phys0_05_026`

However, if you instead added an entirely new physics scheme to the repo, then the new tag should be:

`atmos_phys0_06_000`

Hope that helps, and good luck with tagging!
