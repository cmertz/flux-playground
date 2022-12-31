# flux-playground

`kind` setup to play around with fluxcd.

`rake setup` sets up a kind cluster with:
* fluxcd
* a git server with one config repository
* flux CRs to use the on cluster config repository
* a docker registry to use with kind

`rake config:push` pushes local config to the remote.
The entrypoint is a `Kustomization` CR using everything
under `./config/default/`.

To use images from the docker registry, retag them as
`localhost:5001/IMAGENAME:TAG`, push them and refer
to them in the manifests with the same name.

## required tools in `$PATH`

* kind
* rake
* kubectl
* flux
* ssh
* git
* docker
