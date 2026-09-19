# Natilab Add-ons

A Home Assistant add-on repository. It holds no code and no images - only the
metadata the Supervisor needs to know an add-on exists. The image itself is
pulled from a private registry, named in each add-on's `config.yaml`.

Add it in Home Assistant under Settings -> Add-ons -> Add-on Store -> the
three-dot menu -> Repositories.

## Releasing

A release is two moves that have to stay in step, because the Supervisor asks
the registry for the tag named in `version:`:

1. Push an image tagged `<version>` to the registry.
2. Raise `version:` in `sundial/config.yaml` to match, and push this repository.

Home Assistant then offers the update by itself.
