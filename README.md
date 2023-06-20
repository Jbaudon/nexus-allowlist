# Nexus Allowlist

A script for configuring [Sonatype Nexus Repository Repository OSS](https://github.com/sonatype/nexus-public) to only allow selected packages to be installed from proxy repositories.

Supports creating CRAN and PyPI proxies which allow either all, or only named packages.

## Docker

A [Dockerfile](Dockerfile) and example [docker compose](docker-compose.yaml) configuration demonstrate how to use the script in conjunction with a Nexus OSS container.

## Instructions

### Test deployment

Check and, if you would like, change the following environement variables for the Nexus Allowlist container in [`docker-compose.yaml`](./docker-compose.yaml).

| Environment variable | meaning                                                                                                     |
|----------------------|-------------------------------------------------------------------------------------------------------------|
| NEXUS_ADMIN_PASSWORD | Password for the Nexus OSS admin user (changes from the default on first rune then used for authentication) |
| NEXUS_PACKAGES       | Whether to allow all packages or only selected packages [`all`, `selected`]                                 |
| NEXUS_HOST           | Hostname of Nexus OSS host                                                                                  |
| NEXUS_PORT           | Port of Nexus OSS                                                                                           |

Example allowlist files are included in the repository for [PyPI](allowlists/pypi.allowlist) and [CRAN](allowlists/cran.allowlist).
The PyPI allowlist includes numpy, pandas, matplotlib and their dependencies.
The CRAN allowlist includes cli and data.table
You can add more packages by writing the package names, one per line, in the allowlist files.

Start the Nexus and Nexus Allowlist containers using docker compose

```
docker compose up -d
```

You can monitor the Nexus Allowlist container instance

```
docker compose logs -f allowlist
```

### How it works

The container [command](entrypoint.sh)

1. Ensures that allowlist files `/allowlists/pypi.allowlist` and `/allowlists/cran.allowlist` exist
1. Waits for Nexus OSS to be available at `NEXUS_HOST:NEXUS_PORT`
1. If the Nexus OSS initial password file is present (at `/nexus-data/admin.password`)
  1. Changes the admin password to `NEXUS_ADMIN_PASSWORD`
  1. Runs initial configuration (creates a role, repositories, content selectors, _etc._)
1. Reruns the content selector configuration (which enforces the allowlists) every time either of the allowlist files are modified

### Usage

#### Pip

You can edit `~/.config/pip/pip.conf` to use the Nexus PyPI proxy.
To apply globally edit `/etc/pip.conf`.
For example

```
[global]
index = http://localhost:8081/repository/pypi-proxy/pypi
index-url = http://localhost:8081/repository/pypi-proxy/simple
```

You should now only be able to install packages from the allowlist.
For example,

- `pip install numpy` should succeed
- `pip install mkdocs` should fail

#### R

You can edit `~/.Rprofile`to use the Nexus CRAN proxy.
To apply globally edit `/etc/R/Rprofile.site`.
For example

```
local({
    r <- getOption("repos")
    r["CRAN"] <- "http://localhost:8081/repository/cran-proxy"
    options(repos=r)
})
```
You should now only be able to install packages from the allowlist.
For example,

- `install.packages("data.table")` should succeed
- `install.packages("ggplot2")` should fail
