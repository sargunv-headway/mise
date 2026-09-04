# pipx Backend

pipx is a tool for running Python CLIs in isolated virtualenvs. This isolation helps prevent dependency
conflicts between CLIs, or between a CLI and your Python projects. In short,
this backend lets you add Python CLIs to mise.

To be clear, pipx is not pip, and it is not used to manage Python dependencies in general.
mise is a tool manager, not a dependency manager like pip, uv, or poetry (though you can use mise to install those package
managers). Use the pipx backend to install a CLI like "black", not a library like "NumPy" or "requests".

Somewhat confusingly, the pipx backend defaults to [`uvx`](https://docs.astral.sh/uv/guides/tools/) (uv's equivalent of pipx)
if uv is installed. This mostly means that tools install much faster, but occasionally a tool doesn't work with uvx;
see below for how to disable or configure this.

The pipx backend supports the following sources:

- PyPI
- Git
- GitHub
- HTTP

The code for this is inside the mise repository at [`./src/backend/pipx.rs`](https://github.com/jdx/mise/blob/main/src/backend/pipx.rs).

## Dependencies

This backend requires `uv` (recommended) or `pipx` to be installed.

If you have `uv` installed, mise uses `uv tool install` under the hood, and you don't need `pipx` installed to run commands containing "pipx:".

mise forwards [`minimum_release_age`](/configuration/settings.html#minimum_release_age)
to transitive Python dependency resolution during install. The uv install path uses uv's
`--exclude-newer` flag and requires `uv >= 0.2.22`. The `pipx` fallback passes pip's
`--uploaded-prior-to` flag.

If you need `pipx` for other reasons, you can install it with or without mise.
To install it with mise:

```sh
mise use -g python
pip install --user pipx
```

[Other installation instructions](https://pipx.pypa.io/stable/how-to/install-pipx.html)

## Usage

The following installs the latest version of [black](https://github.com/psf/black)
and sets it as the active version on PATH:

```sh
$ mise use -g pipx:psf/black
$ black --version
black, 24.3.0
```

The version will be set in `~/.config/mise/config.toml` with the following format:

```toml
[tools]
"pipx:psf/black" = "latest"
```

## Python upgrades

If the Python version used by a pipx package changes (whether managed by mise or the system), you may need to
reinstall the package:

```sh
mise install -f pipx:psf/black
```

Or reinstall all pipx packages:

```sh
mise install -f "pipx:*"
```

mise _should_ do this automatically when you run `mise up python`.

### Supported Pipx Syntax

| Description                           | Usage                                                  |
| ------------------------------------- | ------------------------------------------------------ |
| PyPI shorthand latest version         | `pipx:black`                                           |
| PyPI shorthand for specific version   | `pipx:black@24.3.0`                                    |
| GitHub shorthand for latest version   | `pipx:psf/black`                                       |
| GitHub shorthand for specific version | `pipx:psf/black@24.3.0`                                |
| Git syntax for latest version         | `pipx:git+https://github.com/psf/black.git`            |
| Git syntax for a branch               | `pipx:git+https://github.com/psf/black.git@main`       |
| HTTPS with zipfile                    | `pipx:https://github.com/psf/black/archive/18.9b0.zip` |

For GitHub URLs, `latest` resolves to the latest published GitHub Release and falls
back to default-branch HEAD when there are no releases. For other Git URLs, `latest`
tracks default-branch HEAD and resolves it to a concrete commit before installation.
Remote tags are available for explicit version requests.

Other syntax may work but is unsupported and untested.

## Settings

Set these with `mise settings set [VARIABLE]=[VALUE]` or by setting the environment variable listed.

<script setup>
import Settings from '/components/settings.vue';
</script>
<Settings child="pipx" :level="3" />

## Tool Options

The following [tool-options](/dev-tools/#tool-options) are available for the `pipx` backend—these
go in `[tools]` in `mise.toml`.

### `registry_url`

Set the package registry URL mise uses to resolve versions for this tool. The URL must contain a
`{}` placeholder for the package name. This overrides the `pipx.registry_url` setting for this
tool and mise derives the simple index URL used by `uv tool install` or `pipx install` from it.

```toml
[tools]
"pipx:my-tool" = {
  version = "latest",
  registry_url = "https://packages.example.com/pypi/{}/json"
}
```

Simple-index registries can use the package placeholder directly:

```toml
[tools]
"pipx:black" = {
  version = "latest",
  registry_url = "https://libraries.cgr.dev/python/simple/{}/"
}
```

Keep credentials out of `registry_url`; mise rejects URLs containing user information, query
parameters, or fragments because the URL is recorded in `mise.lock` and the install manifest.
Use `.netrc` when authentication must work for both mise's version discovery and the package
manager install. Package-manager-specific credential configuration and `install_env` affect the
install only. Registries can publish different artifacts for the same package version, so changing
`registry_url` causes mise to reinstall that version. Use `uvx_args` or `pipx_args` only when you
need explicit additional indexes or other installer-specific behavior.

### `install_env`

Set environment variables for `uv tool install` or `pipx install`. mise still
sets the tool directory, bin directory, and configured Python package index
variables after applying `install_env`.

```toml
[tools]
"pipx:black" = { version = "latest", install_env = { PIP_TRUSTED_HOST = "pypi.org" } }
```

### `extras`

Install additional components.

```toml
[tools]
"pipx:harlequin" = { version = "latest", extras = "postgres,s3" }
# equivalent array form:
# "pipx:harlequin" = { version = "latest", extras = ["postgres", "s3"] }
# extras also work with Git sources:
# "pipx:psf/black" = { version = "latest", extras = ["jupyter"] }
```

When passing extras inline, use mise's `key=value` tool-option syntax:

```bash
mise use 'pipx:psf/black[extras=jupyter]@latest'
```

For Git repositories whose name differs from the Python distribution name, set `package_name` so
mise can build the requirement used to select extras:

```toml
[tools]
"pipx:owner/repository" = { version = "latest", package_name = "distribution", extras = ["feature"] }
```

### `pipx_args`

Additional arguments to pass to `pipx` when installing the package.

```toml
[tools]
"pipx:black" = { version = "latest", pipx_args = "--preinstall" }
```

### `uvx`

Set to `false` to always disable uv for this tool.

```toml
[tools]
"pipx:ansible" = { version = "latest", uvx = "false", pipx_args = "--include-deps" }
```

### `uvx_args`

Additional arguments to pass to `uvx` when installing the package.

```toml
[tools]
"pipx:ansible-core" = { version = "latest", uvx_args = "--with ansible" }
```
