FROM mcr.microsoft.com/devcontainers/base:ubuntu-24.04

# Must match SHELLCHECK_VERSION in tests/run and .github/workflows/test.yml.
# The suite compares the reported version as an exact string and skips its three
# shellcheck cases on a mismatch; CI fails on any skip.  A drifted pin here is
# therefore green locally and red in CI.
ARG SHELLCHECK_VERSION=v0.11.0

# Empty by default so the committed image matches CI, which runs in the C
# locale.  A machine-local config passes its host's LANG to have that locale
# generated, because setting LANG to an ungenerated locale does not fail -- the
# C library falls back and every program warns on stderr instead.
ARG LOCALE=

# chkstow ships with stow.
#
# info is not optional here, and `man stow` is a trap.
#
# Debian's stow package ships no man page -- only the info manual and an HTML
# split under /usr/share/doc -- and this base image is minimized, so `man stow`
# does not fail: it exits 0 and prints Ubuntu's "this system has been minimized"
# notice.  A caller checking the exit status sees success, and the text reads
# like a page that simply lacks the section being looked for.  On a host whose
# stow came from Homebrew the same command returns a real page, so the
# difference is silent and platform-dependent.
#
# That matters because tool behaviour here is verified against the manual rather
# than from memory, and the findings that mattered most came out of it: that
# stow's built-in ignore list drops .gitignore by basename at any depth, and
# that a conflict aborts before any filesystem modification.  Deprived of a
# reader, a verifier falls back to reasoning -- which is how this repository
# acquired comments asserting protection that did not exist.
#
# Use `info stow`.  Both nodes are present and carry their real content.
RUN apt-get update -qq \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y -qq --no-install-recommends \
        stow \
        xz-utils \
        info \
        locales \
        tzdata \
    && rm -rf /var/lib/apt/lists/*

# Charset is assumed UTF-8: every locale this is meant to carry across is a
# .UTF-8 one, and locale-gen needs the charset named separately from the name.
RUN set -eux; \
    if [ -n "$LOCALE" ]; then \
        echo "$LOCALE UTF-8" >> /etc/locale.gen; \
        locale-gen; \
        locale -a | grep -iE "^${LOCALE%%.*}"; \
    fi

# gh from upstream rather than a devcontainer feature, so `docker build` alone
# produces the complete image.
RUN set -eux; \
    mkdir -p -m 755 /etc/apt/keyrings; \
    curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg \
        -o /etc/apt/keyrings/githubcli-archive-keyring.gpg; \
    chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg; \
    printf 'deb [arch=%s signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main\n' \
        "$(dpkg --print-architecture)" > /etc/apt/sources.list.d/github-cli.list; \
    apt-get update -qq; \
    DEBIAN_FRONTEND=noninteractive apt-get install -y -qq --no-install-recommends gh; \
    rm -rf /var/lib/apt/lists/*; \
    gh --version

# From the upstream release rather than apt, for the reason recorded in
# .github/workflows/test.yml: the archive build reports SC2317 across every
# function the suite dispatches indirectly.
RUN set -eux; \
    case "$(dpkg --print-architecture)" in \
        amd64)   arch=x86_64  ;; \
        arm64)   arch=aarch64 ;; \
        armhf)   arch=armv6hf ;; \
        riscv64) arch=riscv64 ;; \
        *) echo "no shellcheck release for $(dpkg --print-architecture)" >&2; exit 1 ;; \
    esac; \
    url=https://github.com/koalaman/shellcheck/releases/download; \
    curl -fsSL -o /tmp/shellcheck.tar.xz \
        "$url/${SHELLCHECK_VERSION}/shellcheck-${SHELLCHECK_VERSION}.linux.${arch}.tar.xz"; \
    tar -xJf /tmp/shellcheck.tar.xz -C /tmp; \
    install -m 0755 "/tmp/shellcheck-${SHELLCHECK_VERSION}/shellcheck" \
        /usr/local/bin/shellcheck; \
    rm -rf /tmp/shellcheck.tar.xz "/tmp/shellcheck-${SHELLCHECK_VERSION}"; \
    shellcheck --version
