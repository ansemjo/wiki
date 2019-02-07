# Git

## Prevent commits on a branch

You can use a `pre-commit` hook to check the branch name to which you are commiting your changes. If
you want to prevent direct changes to `master` create the following `.git/hooks/pre-commit`:

```
#!/bin/sh

# prevent commits on master
[ "$(git rev-parse --abbrev-ref --symbolic-full-name HEAD)" == "master" ] \
  && { echo "you shall not commit on master"; exit 1; }
```

Make sure the hook script is executable.
