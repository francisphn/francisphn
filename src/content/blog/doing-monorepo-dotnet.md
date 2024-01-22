---
author: Francis Phan
pubDatetime: 2024-01-21T05:24:46.539Z
title: What I've learnt about monorepos with .NET
featured: false
draft: false
tags:
  - devops
  - monorepo
  - git
  - dotnet
description: Monorepos are great for many startups, but they also have caveats.
---

For a year now, I've been involved in Triage-Plus, alongside many great other people. Our codebase,
written primarily in .NET, has naturally been a monorepo since the beginning. Here's how we do it and
what I have learnt out of the process.

If you need a refresher on what a monorepo is, [here's a good article](https://www.atlassian.com/git/tutorials/monorepos).

## We started with a solution

If you use .NET, you will know that .NET is built around the concept of a "solution". A solution can
contain multiple projects, and each project can be a library or an application. This way of structuring
code is by design from the very start of the .NET ecosystem, as opposed to other ecosystems where it's
an afterthought. Therefore, if you use .NET, you will naturally start with a codebase following the
monorepo pattern, and all the native .NET tooling you use assumes you have this pattern.

Other ecosystems like Node/npm or Python/Virtualenv, are not built around the
workspace concept, so you will naturally start with a codebase following the polyrepo pattern.
However, support for workspaces is becoming more common in these ecosystems.

## Microservices and the limits of MSBuild and our pipeline

Our one-solution approach was great. We had a single place to manage all our projects
and did not need to worry about all the problems that come with multiple repositories:

- We didn't need to worry about how to share code between projects. We just added a reference to the
  project we wanted to use.

- We didn't need to worry about versioning dependencies or maintaining a NuGet server. With the
  constant changes we were making to library projects, it would have been a nightmare to manage all
  the different versions of the same library.

- Untold amount of convenience and time saved by being able to search, navigate, refactor, and to
  deliver changes across the entire codebase. A pull request that touches multiple projects is a
  breeze.

All of this allowed us to focus on the product and not worry about the "ops" side of things. As a
single team that was just starting out and owned the entire codebase, this was great.

What I've described above can informally be called the **monorepo pattern**. You put multiple projects
into a single repository and use the native tooling to manage them (in this case, MSBuild or npm).
However, as we scaled, we started to see the limits of purely using the monorepo pattern with just
MSBuild and our pipeline: a single change would force us to test and deploy all the microservices,
even if they were not affected by the change.

## Crossroads No. 1

At this crossroads, we had to make a decision. We could either:

1. Adopt a monorepo tool like Bazel or Nx, or
2. Adopt a polyrepo approach, or
3. Go with a hybrid approach of both.

We read about how our friends at other companies
were doing it like Partly (also based in New Zealand), Canva, and Google. Arguments for and against
monorepos are well documented, so I won't go into them here. Instead, here are the main things we
considered when making our decision.

### 1. Tooling

Having researched into Bazel and Nx, two popular monorepo tools, we thought that they provided a
highly complex abstraction over the dotnet CLI that would require us to pay a hefty tax to adopt:

- Both Bazel and Nx are language-agnostic tools. To support .NET, they use attachable "rules" or
  "plugins" that wraps around the dotnet CLI. This means that there are special commands to do
  things like build, test, serve, etc. The problem with this is that IDEs like Rider are built around
  the dotnet CLI, so we would lose a lot of IDE support, most notably debugging.

- Finally, tools like Bazel and Nx are also more focused on not needing to compile every single
  project when you run `dotnet run`, which was not a problem for us because code compilation was
  imperceptibly fast at our scale, both locally and in the pipeline.

- Instead, we were more conscious on the amount of time it took to run tests, build Docker images,
  and deploy.

Therefore, at the time, we were not ready to make that investment after weighing up the pros and
cons with our then-situation.

### 2. Git is not designed for monorepos

Secondly, Git is not designed for monorepos. Linus Torvalds designed Git to be a _distributed_
version control system.
It requires you to clone the entire repository, which is not ideal for monorepos. This means that
in theory, it will stop scaling at some point.

In practice though, at Canva scale, [Git and Bazel are still working fine enough for them](https://www.canva.dev/blog/engineering/we-put-half-a-million-files-in-one-git-repository-heres-what-we-learned/).
They do need to optimise, and on the occasion run into issues with GitHub, but they have not yet hit
a point where they need a new version control system to manage their monorepo.

For Google, although they never adopted Git, they did hit a point where they needed a new version
control system. Therefore, [they built Piper](https://cacm.acm.org/magazines/2016/7/204032-why-google-stores-billions-of-lines-of-code-in-a-single-repository/fulltext),
a custom version control system that doesn't require you to clone the entire repository and uses the
Paxos algorithm to ensure consistency. But do we think we will ever get to Google scale? I'm an optimist,
but it's a very far-fetched future.

### 3. Open sourcing code and collaboration with clients

Finally, we have a lot of code that we want to open source. As a multitenant SaaS product, we also
allow our clients to write their own code and deploy it to our platform. For this, we need to be
able to give them access to only the code that they need to see. This is not possible with a monorepo.

To solve this problem, Google has two tools called [Repo](https://gerrit.googlesource.com/git-repo/) and [Copybara](https://github.com/google/copybara).
But again, this requires a lot of configuration and maintenance. We soon became aware that some
degree configuration is inevitable whichever way we go, we just need to figure out the least effort
solution to get the most value.

### 4. Code interoperability

One of Canva's reasons for adopting Bazel was the ability to store code in different languages in
the same repository, specifically Java and TypeScript. Bazel or Nx, as neutral build tools (as opposed
to language-specific MSBuild or npm), enable this by nature, and also enable sharing code between these
projects. For us, this was not a problem because we don't seek to interop between different languages
i.e. another benefit of a monorepo tool not applicable to us.

## Being incremental is key

Having considered all of the above, we decided to not adopt a monorepo tool outright. Instead,
we realised that incremental adoption of the monorepo pattern was the best solution for us. This strategy
consists of doing these:

### 1. Have a hybrid monorepo and polyrepo approach

Both monorepos and polyrepos have pros and cons. The fact is, you don't have to buy into one solution 100%,
especially if you do not have a requirement to interop between different languages.

Keeping our internal codebase together, which is 97% C#, is a no-brainer. We got all the practical
benefits of a monorepo listed above for the time being.

For code written in TypeScript and Python, we decided to make separate repositories using these
languages' native build tools (in our case pnpm and pipenv).

Lastly, we made separate repositories for open source and client code. The admin time involved in
maintaining Copybara could yet be justified at our scale.

### 2. Our problem was not about compile time, but with the directed acyclic graph of dependencies

We found that our problem is just about how we can manage the dependencies between our projects.
For example, if we have a project A that depends on project B, and a project C that has no
dependencies, making a change to project A or B should require us to deploy project A, but not project
C. In other words, this is a directed acyclic graph problem.

A custom tool that can build a directed acyclic graph of our projects and determine changes between
commits would be ideal. We could then use this information to determine which projects to re-deploy.
This was all we needed as opposed to a full-blown monorepo tool.

### 3. `dotnet-affected`

Having understood what we need, we were going to build our own DAG analysis tool, but we found that
someone had already done it for .NET on GitHub.

The tool is called [`dotnet-affected`](https://github.com/leonardochaia/dotnet-affected), and it does exactly what we wanted.
And it's been working great for us so far.

## Interlude: using `dotnet-affected`

The tool is well documented, so I won't go into details here. Here's a brief guide:

1. You use NRWL's `nx-set-shas` action to determine the last successful commit on the branch. This is
   the commit that you want to compare against. You can use the `github.base_ref` variable to get the
   branch name. For pull requests, this is the base branch. For pushes, this is the branch you pushed
   to.

2. You use `dotnet-affected` to determine the affected projects between the last successful commit
   and the current commit. You can use the `github.sha` variable to get the current commit SHA.

3. `dotnet-affected` writes the affected projects to a file called `affected.txt` and `affected.proj`.
   You can then use this file to determine which projects to test and re-deploy.

4. Have a conditional on subsequent steps to only run if `affected.txt` is not empty.

Keep in mind, GHA only shallow clones the repository, so you need to set `fetch-depth: 0` in the
`actions/checkout` step.

Here's an example:

```yaml
#  MIT License

#  Copyright (c) Francis Phan.

#  Permission is hereby granted, free of charge, to any person obtaining a copy
#  of this software and associated documentation files (the "Software"), to deal
#  in the Software without restriction, including without limitation the rights
#  to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
#  copies of the Software, and to permit persons to whom the Software is
#  furnished to do so, subject to conditions. You can read the full license at:
#  https://opensource.org/license/mit/

steps:
  - name: Checkout code
    uses: actions/checkout@v4
    with:
      fetch-depth: 0

  - name: Work out last successful commit
    id: setSHAs
    uses: nrwl/nx-set-shas@v4
    with:
      main-branch-name: ${{ github.base_ref }}
      workflow-id: "YOUR_WORKFLOW_YML_FILE_NAME.yml"

  - name: Determine affected projects
    id: dotnet_affected
    run: |
      dotnet tool install dotnet-affected && touch affected.txt

      if dotnet affected -f text traversal \
        --from "${{ steps.setSHAs.outputs.base }}" \
        --to "${{ github.sha }}"; then
       if [ -s affected.txt ]; then
         cat affected.txt
         echo "Number of affected projects: " $(wc -l < affected.txt)
         echo "has_output=true" >> $GITHUB_OUTPUT
       else
         echo "No affected projects."
         echo "has_output=false" >> $GITHUB_OUTPUT
       fi
      elif [ $? -ne 166 ]; then
       echo "Exiting, error occurred"
       exit 1
      else
       echo "No affected projects."
       echo "has_output=false" >> $GITHUB_OUTPUT
      fi

  - name: Test affected projects
    if: steps.dotnet_affected.outputs.has_output == 'true'
    run: dotnet test affected.proj
```

## This is not an entirely scalable solution, but it leaves room for growth

For a time, this worked for us because the time it takes to recompile our entire solution was
fast both locally and in the pipeline (i.e. the problem Bazel and Nx focus on solving).

However, as we've grown, we have been able to perceive that the time it takes to recompile our
entire solution is slightly increasing (but still not a major problem). When it really is at odds
with our patience, we will eventually consider using a monorepo tool.

The important thing is that we have left room for growth. We already use the monorepo pattern, so
we can easily adopt a monorepo tool like Bazel, Nx or Pants without having to majorly refactor our codebase.

## Conclusion

In summary, adopting a monorepo requires consideration of many factors. I recommend studying
the pros and cons of it, and see which ones apply to your situation. Put both options on a mental
weighing scale and see which way it leans.

It's also important to note that even if you've decided on an approach or use a hybrid approach like
ours, like any conscious decision in tech or in life, you will eventually have to accept some level
of trade-offs (unless you're Google and you have the resources to build your own solution).

If you do decide that a monorepo is suitable for you, I also highly recommend an incremental approach.
Adopt the monorepo pattern first, and then adopt a monorepo tool as you scale. This way, you can get
the benefits of a monorepo without the overhead of a monorepo tool, and when the time comes, it
makes it easier to adopt a monorepo tool.
