# nx-deps-not-running

Sample repository to show that Nx isn't firing all `targetDependencies`.

# structure

There are two projects:

- one
- two

Via an implicit dependency, project "two" depends on project "one".

Project "one" has a `generate-sources` target. Via `nx.json`, a project's `generate-sources` target depends on its dependencies' `generate-sources` _and_ `build` targets. (The fact that it depends on both is critical to this issue.)

Project "two" has a single `build` target but does _not_ have a `generate-sources` target (this is also critical). It produces output that project "one" requires when running `generate-sources`.

## instructions

1. `npm i`
1. `rm -rf ./dist`
1. `nx run two:generate-sources`

## expected

Before the the `generate-sources` target for project "two" runs, the `build` target for project "one" should run.

## actual

The `build` target for project "one" never runs.

# summary

This used to work and regressed in version 13.10.

As far as I can tell, there are two things working together to cause the problem. First, `nx.json` has _two_ target dependencies for `generate-sources`:

```json
  "targetDependencies": {
    "generate-sources": [
      {
        "target": "generate-sources",
        "projects": "dependencies"
      },
      {
        "target": "build",
        "projects": "dependencies"
      }
    ]
  }
```

That, coupled with [these lines](https://github.com/nrwl/nx/blob/b6617105f32ba61e1e40f6ff9f96fc0ba333c7fb/packages/nx/src/tasks-runner/run-command.ts#L398-L400) of code causes the process to short-circuit before the dependent targets are run.

This is what happens:

1. Nx looks for targets that should run before `two:generate-sources`.
1. It finds project "one" and checks if it has a `generate-sources` target. It does not, _but Nx flags the whole project as "seen"_.
1. It finds project "one" and checks if it has a `build` target. It does, but before `addTasksForProjectDependencyConfig()` is called, it checks if the project has already been seen. It has, so it short-circuits. Therefore, `one:build` never runs.

This happens because there are two blocks within `targetDependencies`. Once that first block executes, dependent projects are marked as "seen", and the second block never fires. Also, adding `generate-sources` to project "one" doesn't solve the problem. That target fires, but the project is still marked as "seen", which means that `build` target never fires.

The "seen" logic was introduced with the circular dependency check, which makes sense, but this is not a circular dependency. Commenting out out [the short-circuit](https://github.com/nrwl/nx/blob/b6617105f32ba61e1e40f6ff9f96fc0ba333c7fb/packages/nx/src/tasks-runner/run-command.ts#L399) fixes the problem, but it almost certainly breaks the circular dependency checking, so I'm not sure what the correct fix is.
