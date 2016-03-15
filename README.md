Say you have:
* an app called `app`
* `app` has a devDep on `dev-dep@1.0.0`
* `app` depends on `local-dep` via `npm link` or otherwise via symlink
* `local-dep` has a devDep on `dev-dep@2.0.0`

In some cases `npm ls` (in `app`'s root dir) will display the info for `local-dep`'s dependency on `dev-dep` instead of `app`'s. It's non-deterministic in the sense that it depends on the name of the linked package (e.g. results may be different if it's `a-local-dep` vs `z-local-dep`). If it comes after the name of the main package in the sort implemented by `npm`, then the linked package's devDep information will overwrite the main package's in the output.

With `node@5.6.0`, `npm@3.8.1`, on Ubuntu 12.

```sh
git clone {{this repo}}
cd {{this repo dir}}
sh setup.sh

cd ./linker
npm ls
# ^^ Note that the list contains `react-dom@15.0.0-rc.1`.

# Note that this will come before `linker` when the dep tree is flattened and
# sorted:

ln -s $PWD/../linked node_modules/linked

# Use this instead if you want to repro with `npm link`:
# npm link ../linked

npm ls
# ^^ Note that the list still contains `react-dom@15.0.0-rc.1`.

# Note that this will come after `linker` when the dep tree is flattened and
# sorted:

ln -s $PWD/../z-linked node_modules/z-linked

# Use this instead if you want to repro with `npm link`:
# npm link ../z-linked

npm ls
# ^^ Note that the list now contains `react-dom@0.14.7`.
# Note also the following, even though react-dom@15.0.0-rc.1 doesn't show up in
# the tree:
# npm ERR! peer dep missing: react@^15.0.0-rc.1, required by react-dom@15.0.0-rc.1

npm rm z-linked
npm ls
# ^^ Note that the list is back to containing `react-dom@15.0.0-rc.1`.
```

Key action is happening here:

* [mutate-into-logical-tree.js#L27-L35]( https://github.com/npm/npm/blob/580f88a8183375a9c045529f10bf1aaea75f7bdd/lib/install/mutate-into-logical-tree.js#L27-L35)

* [mutate-into-logical-tree.js#L71](
https://github.com/npm/npm/blob/580f88a8183375a9c045529f10bf1aaea75f7bdd/lib/install/mutate-into-logical-tree.js#L71)
