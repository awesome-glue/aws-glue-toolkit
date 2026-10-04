# AWS Glue Toolkit

A streamlined command-line utility designed to simplify the AWS Glue development lifecycle.

> **Disclaimer:** This is an independent, community-maintained project. It is **not** affiliated with, endorsed by, or sponsored by Amazon Web Services (AWS). AWS, AWS Glue, and related marks are trademarks of Amazon.com, Inc. or its affiliates.

## Installation

Requires Python 3.11+. Install [Docker](https://docs.docker.com/get-docker/) on the host for `run`, `test`, and `gtk build --mode container` (Glue image pulled on first use). Default `gtk build` (`--mode host`) uses local pip. Installing the package adds the `gtk` command and a compatible `pip` release.

```bash
pip install aws-glue-toolkit
```

## Example

A Glue job is a directory with a `pyproject.toml` and a source tree. Given this layout:

```
my-glue-job/
  pyproject.toml
  src/
    __main__.py
  tests/
    test_example.py
```

configure the job in `pyproject.toml`:

```toml
[project]
name = "my-glue-job"
version = "0.1.0"
dependencies = ["pandas>=2"]

[tool.aws-glue-toolkit]
glue_version = "5.1"
source = "src"
script = "__main__.py"
tests = "tests"
```

then run:

```bash
cd my-glue-job
gtk build .
gtk run .
gtk test .
```

Use `gtk build` to verify dependencies against Glue runtime pins.

## Configuration

Each job is a directory containing `pyproject.toml`. `gtk` reads the `[project]` and `[tool.aws-glue-toolkit]` fields in the table below; other keys in the file are skipped. The `source` directory and entry `script` must exist before `gtk` runs.

| Field | Required | Default | Description |
| --- | --- | --- | --- |
| `project.name` | yes | — | Job name; used in artifact file names |
| `project.version` | no | `0.0.0` | Job version; used in artifact file names |
| `project.dependencies` | no | `[]` | Direct dependencies as PEP 508 strings (PyPI, git/VCS, or `file:`) |
| `tool.aws-glue-toolkit.glue_version` | yes | — | Glue release; bundled pins for `5.0` and `5.1` |
| `tool.aws-glue-toolkit.source` | yes | — | Source directory, relative to the job root |
| `tool.aws-glue-toolkit.script` | yes | — | Entry script, relative to `source` |
| `tool.aws-glue-toolkit.tests` | no | `tests` | Test directory, relative to the job root |
| `tool.aws-glue-toolkit.dependencies` | no | `[]` | Extra PEP 508 strings merged after `project.dependencies`; recommended for relative `file:` paths |

### Pip configuration

For container pip (`build --mode container`, and dependency install on `run`/`test`), `gtk` snapshots the host’s effective [pip configuration](https://pip.pypa.io/en/stable/topics/configuration/) (files + `PIP_*`, env wins) into a temporary `pip.conf`, mounts it, and sets `PIP_CONFIG_FILE`. The snapshot includes settings that apply inside the container (index URLs, extra indexes, and similar). Default `--mode host` uses host pip configuration as-is.

```bash
# ~/.config/pip/pip.conf  or:
export PIP_EXTRA_INDEX_URL="https://my.company/simple"
gtk build .
```

## Commands

`[JOB-DIR]` is the job directory path as the first positional argument (default `.`) on **`build`**, **`run`**, and **`test`**.

| Command | Usage |
| --- | --- |
| build | `gtk build [JOB-DIR] [--mode host\|container]` |
| run | `gtk run [JOB-DIR] [--platform native\|worker] [args...]` |
| test | `gtk test [JOB-DIR] [--platform native\|worker] [pytest args...]` |

For `run` / `test`: job dir mounted at `/home/hadoop/workspace`; non-empty job dependencies install into an ephemeral `PYTHONPATH` target (Glue pins as constraints); stdio pass through; exit code is the container command’s (or pip’s if install fails). Same dependency forms as `build`.

### build

Writes a dependencies zip (`.py` under `source`, except the entry script) and a gluewheels zip. Resolves job dependencies against Glue runtime pins and packages wheels for deployment.

- **`--mode host` (default):** host packaging with local pip. Path/VCS via `pip wheel --no-deps`; other packages via `pip download --platform --only-binary=:all:`, or sdist→wheel on the host when no compatible wheel exists. Wheels must be `any` or the Glue `pip_platform` (use `--mode container` for compiled packages that need a worker-arch build). VCS needs network and `git` on the host.
- **`--mode container`:** `pip wheel` in the Glue image (worker-arch). On ARM hosts this needs QEMU (or similar) unless you use `--mode host` for portable wheels. Use this mode in continuous integration when you want the same pip environment as worker-arch packaging.

Path deps must be installable packages (`pyproject.toml` or `setup.py`); loose job modules belong under `source`. Both zips are always written; gluewheels omits packages already pinned on the image at the same version.

#### Overriding image-bundled packages

Glue workers ship many Python libraries at fixed versions (see bundled runtime metadata). By default those versions are pip constraints. To ship a **different** version of a package that is already on the image, add it as a **direct** dependency (in `[project].dependencies` or merged `[tool.aws-glue-toolkit].dependencies`). That name is removed from pip constraints so your spec wins; `build`, `run`, and `test` all use the same rules. If the resolved version still matches the image pin, the wheel is omitted from gluewheels as usual. Other image pins keep constraining transitives unless you list those packages too.

Example for Glue 5.1 (image ships `pandas==2.3.3`):

```toml
[project]
dependencies = ["pandas==2.2.3"]
```

| File | Glue parameter | Contents |
| --- | --- | --- |
| `{name}-{version}.dependencies.zip` | `--extra-py-files` | `.py` files under `source`, except the entry script |
| `{name}-{version}.gluewheels.zip` | `--additional-python-modules` (Glue 5.0+) | `wheels/requirements.txt` and `*.whl` per [AWS Glue Appendix A](https://docs.aws.amazon.com/glue/latest/dg/aws-glue-programming-python-libraries.html) |

#### Dependency forms

| Form | Example | Notes |
| --- | --- | --- |
| PyPI | `pydantic==2.13.4` | Version pins or ranges |
| Path (in job) | `my-lib @ file:./libs/my-lib` | Built into a wheel at build time |
| Path (monorepo) | `shared @ file:../packages/shared` | Container mode: mounted into the image; host mode: host path |
| Git (HTTPS) | `tool @ git+https://github.com/org/tool.git@v1` | Needs network + `git` |

For monorepo path dependencies, list relative `file:` specs under `[tool.aws-glue-toolkit].dependencies` (merged after `[project].dependencies`):

```toml
[project]
name = "my-glue-job"
version = "0.1.0"
dependencies = ["pandas>=2"]

[tool.aws-glue-toolkit]
glue_version = "5.1"
source = "src"
script = "__main__.py"
dependencies = [
  "my-lib @ file:./libs/my-lib",
  "shared @ file:../packages/shared",
]
```

You may still list relative `file:` specs under `[project].dependencies`; `gtk` accepts either location.

### run

`spark-submit` on the entry script. Passes `--JOB_NAME` from `project.name` unless overridden. Extra `--key value` tokens after `[JOB-DIR]` go to `getResolvedOptions` (pass an explicit `[JOB-DIR]` when using `.`). Shuts down the Spark driver so the container returns.

- **`--platform native` (default):** Docker runs the official multi-arch Glue image for the host (arm64 on Apple Silicon / ARM CI). Local CPU may differ from cloud Glue workers (amd64).
- **`--platform worker`:** Docker uses `worker_docker_platform` from runtime metadata (today `linux/amd64`). On ARM this needs QEMU (or an amd64 machine).

### test

`python3 -m pytest`. The configured `tests` directory must exist before `gtk test` runs (exit code 78 if it is missing). `PYTHONPATH` includes `source` (and the install target when deps are present). Default target is the configured `tests` dir; if the first forwarded token starts with `-`, that dir is still passed first (`gtk test . -v`); if it is a path, only those tokens go to pytest.

Same `--platform native|worker` as `run`. Prefer `--platform worker` (or amd64 CI / real Glue) when validating native libs or Spark bits that can differ by CPU.

### Exit codes

Documented failures use Berkeley `sysexits`-style codes in the 64–78 band.

| Code | Meaning |
| --- | --- |
| 65 | Pip or dependency resolution failure (`build`, `run`, or `test`) |
| 66 | `pyproject.toml` missing or unreadable |
| 69 | Docker not available or could not be started |
| 70 | **`build` only:** zip or other packaging failure on the host |
| 78 | Invalid `pyproject.toml`, job layout (missing source, script, or tests directory), unsupported Glue version, or invalid dependency |

Success is `0`; unhandled errors are `1`. **`build`** uses the codes above. **`run`** and **`test`** return the container command’s exit code when pip install and the main command succeed (pytest exit code `1` means failing tests).

### Stability

The stable interface is the **`gtk` command-line interface**: command names, flags, and documented exit codes (table above).

## Architecture

Two-layer module layout (core and shell) is documented in
[docs/architecture.md](docs/architecture.md).

## Contributing

Contributions are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) for setup and workflow. Participants are expected to follow the [Code of Conduct](CODE_OF_CONDUCT.md). Report security vulnerabilities privately via [SECURITY.md](SECURITY.md).

## License

This project is licensed under the MIT License — see the [LICENSE](https://github.com/HH-MWB/aws-glue-toolkit/blob/main/LICENSE) file for details.
