# Contributing

## Checks (run before every push; CI runs the same)

```sh
gofmt -l .        # must print nothing
go vet ./...
go test ./...
```

CI skips the Go steps until `go.mod` exists.

## Pull requests

`main` is protected: changes land through pull requests with green CI.

1. Pick an issue. Branch `<type>/<short-desc>` (e.g. `feat/config`), Conventional
   Commit PR title, `Closes #N` in the description.
2. Open the PR as a draft and iterate while CI runs.
3. Mark it ready (`gh pr ready`). That triggers an automated Claude review.
4. Address every review comment: fix it, or reply briefly why not.
5. Squash-merge once CI is green and the review is addressed.
