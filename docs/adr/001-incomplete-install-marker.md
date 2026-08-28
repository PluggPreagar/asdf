# ADR 001: Use atomic directory rename with marker file to signal incomplete installs

## Status

Accepted

## Context

Due to the initial implementation of asdf all plugin versions must be installed into a subdirectory inside the asdf installs directory. When an installation is terminated by `SIGTERM`, `SIGKILL`, a dropped network connection, or loss of power, the installation directory persists, even though the installation itself was interrupted and is likely in an incomplete or broken state. When a user later runs commands that check for installed versions asdf incorrectly treats these incomplete installations as installed. This happens because asdf considers every directory inside of `$ASDF_DATA_DIR/installs` as a valid installation. This leads to confusing behavior and errors the user may be unable to trace back to the root cause.

## Decision

We will implement a mechanism to mark incomplete version installation directories. The mechanism will be robust and prevent any type of failure from resulting in an installation that asdf treats as valid when it is incomplete.

Here is how the install process will work:

1. Before beginning an installation asdf will check if the installation directory already exists with a `.incomplete` marker file. If it does, the directory is removed to clean up any stale incomplete installations.
2. Instead of creating a directory named `$ASDF_DATA_DIR/installs/<tool>/<version>` at the start of the install process we will create a directory named `$ASDF_DATA_DIR/temp/<tool>-<version>`, then inside it create an empty `.incomplete` marker file. This directory will then be renamed to `$ASDF_DATA_DIR/installs/<tool>/<version>`. Doing this ensures that the directory always starts with a `.incomplete` file in it. If the installation gets interrupted before the `.incomplete` marker file is created it would only exist in the temp directory and would never have been moved.
3. The plugin's download and install callbacks are invoked, along with any pre-download and pre-install hooks. If the install callback fails the installation directory is removed.
4. When the installation is finished asdf removes the `.incomplete` marker file.

Additionally, signal handlers will be registered for `SIGINT` and `SIGTERM` before installation that will trigger removal of the install directory.

Commands that list installed versions or check for an installed version do this by reading directories. Now there will be an additional check for the `.incomplete` file inside of each directory.

## Consequences

### Positive

* Signal interruption (Ctrl+C) properly cleans up partial installations
* No manual cleanup required for failed installations
* Clear distinction between complete and incomplete installations
* Users no longer encounter confusing errors from partial installations
* Backward compatibility is maintained, all plugins will continue to work

### Negative/Neutral

* Increase in installation process complexity with temporary directory and marker file
* `.incomplete` marker file will be present during installation process

## Related Issues

* https://github.com/asdf-vm/asdf/issues/2184
* https://github.com/asdf-vm/asdf/issues/2036
