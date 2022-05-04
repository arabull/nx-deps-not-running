# nx-deps-not-running

Sample repository to show that Nx isn't firing all `targetDependencies`.

# structure

There are two projects:

- one
- two

Via an implicit dependency, project "two" depends on project "one".

Project "one" has a `build` target. Via `nx.json`, a project's `build` target depends on its dependencies' `generate-sources` _and_ `build` targets. (The fact that it depends on both is critical to this issue.) The project's `build` target produces output that project "two" requires.

Project "two" has a single `build` target but does _not_ have a `generate-sources` target (this is also critical). It looks for the output from the `build` target run by project "one".

## instructions

1. `npm i`
1. `rm -rf ./dist`
1. `nx run two:build`

## expected

Before the the `build` target for project "two" runs, the `build` target for project "one" should run.

## actual

The `build` target for project "one" never runs.

# summary

This used to work and regressed in version 13.10.

As far as I can tell, there are two things working together to cause the problem. First, `nx.json` has _two_ target dependencies for `build`:

```json
  "targetDependencies": {
    "build": [
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

The name of the first target is irrelevant to the issue at hand, but the fact that there _is_ a first target is critical.

A partial fix went in with https://github.com/nrwl/nx/pull/9942. That took care of the situation where a dependent project had two targets but only the first ran. That's not the case here. In this case, the dependent project _only has the second target_. If I add a `generate-sources` target to project "one", the problem goes away (thanks to the https://github.com/nrwl/nx/pull/9942 fix). But adding that target shouldn't be a requirement to run subsequent targets.
