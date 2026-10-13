# PBT-03: Path containment security

## Target

- Function: `documents.file_handling.validate_path_in_root(path, root)`
- Risk: traversal, prefix confusion, or symlink resolution may let a path
  escape the configured storage root.
- Expected result: return only when `path.resolve()` is inside
  `root.resolve()`; otherwise raise `UnsafeFilePathError`.

## Property model

Let `R = root.resolve()` and `P = path.resolve()`. The reference oracle is
`P.is_relative_to(R)`.

| ID         | Invariant                                                  | Generator strategy                                                                   | Oracle                                                                 |
| ---------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------------ | ---------------------------------------------------------------------- |
| PBT03-INV1 | Every generated descendant of `root` is accepted.          | Lists of 1-5 bounded safe path components.                                           | The function returns and `P.is_relative_to(R)` is true.                |
| PBT03-INV2 | Every generated path resolving outside `root` is rejected. | Traversal depth 1-8, nested prefixes 0-5, and sibling names sharing the root prefix. | `P.is_relative_to(R)` is false and `UnsafeFilePathError` is raised.    |
| PBT03-INV3 | Symlink acceptance depends on the resolved target.         | Bounded generated target names for internal and external symlinks.                   | Internal targets return; external targets raise `UnsafeFilePathError`. |

## Preconditions and bounds

- Temporary roots are created by pytest and never use production paths.
- Components use a portable ASCII alphabet and a `p_` prefix to avoid
  reserved or ambiguous names.
- Traversal depth and component counts are bounded for fast shrinking and
  deterministic CI runtime.
- Symlink properties skip only where symlink creation is unavailable.

## Reproduction and evidence

```bash
uv run --python 3.12 --dev --frozen pytest \
  src/documents/tests/test_path_security.py -k TestPathSecurity -v \
  --hypothesis-seed=20261008 --hypothesis-show-statistics
```

The `PBT-03 path security` job uploads `pbt-03-run.log` as the
`pbt-03-evidence` artifact. Hypothesis reports the minimized counterexample
when an invariant fails.

## Limitations

- OS-specific parsing outside `pathlib.Path.resolve()` is out of scope.
- Symlink checks require host support and permissions.
- The suite verifies path containment, not filesystem ACLs.
