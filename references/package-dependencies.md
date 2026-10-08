# Third-party dependencies of shared packages

## Browser asset repository and file contracts

`skeeks/yii2-sx` supplies Underscore through `npm-asset/underscore: ^1.13.8`
and its `Undescore` compatibility class publishes `@npm/underscore/underscore-min.js`.
Keep the class name for existing `Core` and project consumers. The UMD build
provides the global `_` required by the SkeekS browser runtime; ESM and Node
builds are not substitutes for a normal script tag. In npm 1.13.8,
`underscore-min.js` and `underscore-umd-min.js` are byte-identical; retain the
historical filename instead of changing consuming script conventions.

When changing an asset dependency, verify the versions returned by the actual
Composer repository with `show --available --no-cache`, not `show --all`:
the latter also includes installed versions which the repository may no longer
offer. A GitHub tag alone does not prove Composer availability. Verify the
archive's browser filename and the resolved AssetBundle graph as well as the
version constraint. The Bower Underscore feed was verified to offer only up to
1.13.0 while the npm feed offered 1.13.8; the Bower 1.13.0 archive lacked the
historical `underscore-min.js` file and broke the shared runtime after update.

## Advisory blocking

Composer 2.9+ refuses, by default (`policy.advisories.block`), to select any
package version covered by a security advisory during `composer update`.
`--no-audit` and `-W` do not lift it; `composer install` from an existing lock
still works. A shared package whose constraint allows only affected versions
therefore breaks dependency resolution on every site that installs it,
including the site self-update flows driven by `cms-hosting`.

Rules for `skeeks/*` packages:

- Every third-party constraint must allow at least one version without
  advisories. Raise the lower bound to the first fixed release
  (for example `"phpseclib/phpseclib": "^3.0.57"`).
- Check transitive pins too: an abandoned wrapper that pins an old major
  blocks resolution as surely as a direct requirement. Prefer replacing a
  thin, unmaintained wrapper with a small helper in the owning package over
  keeping the wrapper.
- Verify on a copy of a consuming project's lock: partial
  `composer update <changed packages>` plus `composer audit --locked`
  must report no advisories. If the advisories API is unreachable Composer
  only warns and the check proves nothing; retry until data is fetched.
- Keep the production lock rule of the consuming project: update only the
  changed packages from the production lock, never a full `composer update`.

Example: `skeeks/cms-hosting` 2.6.9 replaced phpseclib 2 and
`hguenot/yii2-gsftp` (pinned `~2.0`) with phpseclib 3 helpers; its
`SSH-CLIENT.md` owns the SSH details.
