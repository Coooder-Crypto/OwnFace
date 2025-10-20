# OwnFace

## Repository Layout

This meta repository aggregates the three main OwnFace projects as Git submodules:

- `backend`: https://github.com/Coooder-Crypto/OwnFace-backend
- `frontend`: https://github.com/Coooder-Crypto/OwnFace-frontend
- `contract`: https://github.com/Coooder-Crypto/OwnFace-contract

Each submodule keeps its own Git history and release workflow so the services can evolve independently.

## Cloning

```sh
git clone --recurse-submodules https://github.com/Coooder-Crypto/OwnFace.git
```

If the repository was cloned without the `--recurse-submodules` flag, run:

```sh
git submodule update --init --recursive
```

## Updating Submodules

To pull the latest changes for every project:

```sh
git submodule update --remote
```

After updating, commit the submodule pointer changes in this repository.
