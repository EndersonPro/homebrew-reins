# homebrew-reins

Homebrew tap, Scoop bucket, and release install script for [reins-hook](https://github.com/EndersonPro/reins).

## Install

```sh
brew tap EndersonPro/reins
brew install reins-hook
```

## Update

```sh
brew update && brew upgrade reins-hook
reins-hook install
brew services restart reins-hook
```

The restart is not optional — see [docs/UPDATING.md](docs/UPDATING.md) for why,
how to verify the new version is actually serving, and the two traps that make
a successful `brew upgrade` look like it did nothing.
