# Bashistrano Changelog

Bashistrano versions are following [SemVer](http://semver.org).

## [`master`]

[`master`]: https://github.com/team-supercharge/bashistrano/compare/v1.2.0...HEAD

* Latest contributions

---

## [`1.2.0`]

[`1.2.0`]: https://github.com/team-supercharge/bashistrano/compare/v1.1.3...v1.2.0

### Breaking changes:

* None

### New features:

* Add the ability to specify SSH port for servers

### Fixes

* None

---

## [`1.1.3`]

[`1.1.3`]: https://github.com/team-supercharge/bashistrano/compare/v1.1.2...v1.1.3

### Breaking changes:

* None

### New features:

* None

### Fixes

* Avoid unbound variable error in case of empty parameter lists
* Fix opening SSH tunnels

---

## [`1.1.2`]

[`1.1.2`]: https://github.com/team-supercharge/bashistrano/compare/v1.1.1...v1.1.2

### Breaking changes:

* None

### New features:

* None

### Fixes

* Fix image upload checking by running on every node one-by-one
* Skip pull_images and push_images steps when no images configured

---

## [`1.1.1`]

[`1.1.1`]: https://github.com/team-supercharge/bashistrano/compare/v1.1.0...v1.1.1

### Breaking changes:

* None

### New features:

* None

### Fixes

* Set proper flags for failing commands to exit Bashistrano

---

## [`1.1.0`]

[`1.1.0`]: https://github.com/team-supercharge/bashistrano/compare/v1.0.0...v1.1.0

### Breaking changes:

* None

### New features:

* Check Docker image IDs to prevent unnecessary uploading
* Capture output of `run_locally` and `run_remotely` commands

### Fixes

* None

---

## `1.0.0` (2018-01-28)

### Breaking changes:

* None

### New features:

* Initial version containing
  - Stages
  - Phases & steps
  - Hooks
  - Docker image handling
  - Using sshpass

### Fixes

* None
