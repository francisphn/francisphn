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

For a year now, I've been involved in TriagePlus, alongside many great other people. Our codebase,
written primarily in .NET, has naturally been a monorepo since the beginning. Here's how we do it and
what I have learnt out of the process.

If you need a refresher on what a monorepo is, [here's a good article](https://www.atlassian.com/git/tutorials/monorepos).

## .NET encourages a monorepo mental model

Unlike many other ecosystems (think Node/npm or Virtualenv), .NET gives you a "solution" that can
contain multiple projects. Everything in .NET is built around this concept, as opposed to
other ecosystems where it's an afterthought. This mental model makes it easy to both do monorepos and
promote the microservice pattern.

If you're considering .NET for your next enterprise-scale, cloud-based, modern codebase,
this is one of its strongest selling points.

## Scaling from the solution

We started with that one solution (and still have only one till this day), and it was great. We had a single place to manage all our projects and
did not need to worry about all the problems that come with multiple repositories:

- We didn't need to worry about how to share code between projects. We just added a reference to the
  project we wanted to use.

- We didn't need to worry about versioning dependencies or maintaining a NuGet server. With the constant changes we were making to
  library projects, it would have been a nightmare to manage all the different versions of the same
  library.

- Untold amount of convenience and time saved by being able to search, navigate, refactor, and to
  deliver changes across the entire codebase. A pull request that touches multiple projects is a
  breeze.

All of this allowed us to focus on the product and not worry about the "ops" side of things. As a single team
that was just starting out and owned the entire codebase, this was a great way to start.

As we scaled into maintaining multiple microservices, we started to see the limits. A single change
would force us to test and deploy all the microservices, even if they were not affected by the change.

## Considerations in adopting a monorepo

At this point, after doing some research, we made a highly conscious decision to "officially" adopt the
monorepo pattern moving forward, especially after reading about my friends at other companies were doing it like Partly (also based in New Zealand),
Canva, and Google. Arguments for and against monorepos are well documented, so I won't go into them here. Instead, here are the main
things we considered when adopting the monorepo pattern.

### Tooling: Bazel and Nx

Having researched into Bazel and Nx, we thought that they provided a highly complex abstraction over the dotnet CLI. It would
change everything to do with how we build, test, debug, deploy and reference dependencies. Bazel and Nx requires a lot of configuration
to get right and maintain, in the pipeline and in IDEs. This takes a high level of buy-in and amount of effort to get right.
Tools like Bazel and Nx are also more focused on not needing to build every project\_ when you run `dotnet run`, which was
not a problem for us because code compilation was imperceptibly fast. Instead, we were more conscious on the amount of time
it took to run tests, build Docker images, deploy. At the time, we were not ready
to make that investment.

### Git is not designed for monorepos

Git is not designed for monorepos. Linus Torvalds designed Git to be a _distributed_ version control system.
It requires you to clone the entire repository, which is not ideal for monorepos. This means that at some point,
it will stop scaling.

Google never adopted Git but to solve this problem, they built Piper, a custom version control system
that doesn't require you to clone the entire repository and use the Paxos algorithm to ensure consistency.
See [this article](https://cacm.acm.org/magazines/2016/7/204032-why-google-stores-billions-of-lines-of-code-in-a-single-repository/fulltext) for more details.

That said, at Canva scale, they still use Git alongside Bazel. They have a
[great article](https://www.canva.dev/blog/engineering/we-put-half-a-million-files-in-one-git-repository-heres-what-we-learned/)
on how they solve performance issues with Git. My friends at Canva did tell me that they, do still, however,
face issues specifically with GitHub. That said, GitHub has made considerable improvements to their monorepo support,
including introducing a merge queue feature so that two pull requests that touch the same file can be merged in the correct order.

### Open sourcing code and collaboration with clients

We have a lot of code that we want to open source. As a multitenant SaaS product, we also allow our clients to write their own code and deploy it to our platform.
For this, we need to be able to give them access to only the code that they need to see. This is not possible with a monorepo.

To solve this problem, Google has a tool called [Repo](https://gerrit.googlesource.com/git-repo/) and a tool called [Copybara](https://github.com/google/copybara).
But again, this requires a lot of configuration and maintenance. We became aware that configuration is inevitable while figuring out which is the best approach for us
that requires the least amount of effort.

### Languages other than C#

We have a lot of code written in other languages, such as Python and TypeScript. But we don't do interop between these languages, so we don't need to worry about
sharing code between them. The main problem is that 97% of our code is in C#.

## Our decision

### Don't be ideological

We decided to not be ideological about it. We decided to adopt the monorepo pattern for our internal codebase, but not for our open source codebase.
The client codebase is independent of ours, so the benegits of a monorepo don't apply.

### Directed acyclic graph

We found that our problem is just about how we can manage the dependencies between our projects.
For example, if we have a project A that depends on project B, and a project C that has no dependencies,
making a change to project A or B should require us to deploy project A, but not project C. In other words,
this is a directed acyclic graph problem.

A custom tool that can build a directed acyclic graph of our projects and determine changes between commits
would be ideal. We could then use this information to determine which projects to deploy. This was all we needed
as opposed to a full-blown monorepo tool.

### `dotnet-affected`

We were going to build our own tool, but we found that someone had already done it. We found a tool called
[`dotnet-affected`](https://github.com/leonardochaia/dotnet-affected) that does exactly what we wanted.
It's been working great for us so far.

## Using `dotnet-affected`

The tool is well documented, so I won't go into details here. Here's a brief guide:

1. You use NRWL's `nx-set-shas` to determine the last successful commit on the branch. This is
   the commit that you want to compare against. You can use the `github.base_ref` variable to get the
   branch name. For pull requests, this is the base branch. For pushes, this is the branch you pushed to.

2. You use `dotnet-affected` to determine the affected projects between the last successful commit
   and the current commit. You can use the `github.sha` variable to get the current commit SHA.

3. `dotnet-affected` writes the affected projects to a file called `affected.txt` and `affected.proj`.
   You can then use this file to determine which projects to test and re-deploy.

4. Have a conditional on subsequent steps to only run if `affected.txt` is not empty.

Keep in mind, GHA only shallow clones the repository, so you need to set `fetch-depth: 0` in the `actions/checkout` step.

Here's an example:

```yaml
#  Copyright (c) 2024 Francis Phan.

#  Permission is hereby granted, free of charge, to any person obtaining a copy
#  of this software and associated documentation files (the "Software"), to deal
#  in the Software without restriction, including without limitation the rights
#  to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
#  copies of the Software, and to permit persons to whom the Software is
#  furnished to do so, subject to the following conditions:
#
#  The above copyright notice and this permission notice shall be included in all
#  copies or substantial portions of the Software.
#
#  THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
#  IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
#  FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
#  AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
#  LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
#  OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
#  SOFTWARE.

---
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

      if dotnet affected -f text traversal --from "${{ steps.setSHAs.outputs.base }}" --to "${{ github.sha }}"; then
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

## This is not an entirely scalable solution

Right now, this works for us because the time it takes to recompile our entire solution is still fast both locally and in the pipeline.
However, as we grow, we have been able to perceive that the time it takes to recompile our entire solution is slightly increasing.

Currently though, we are not at the scale where we need to worry about this. The time to recompile our entire solution is still
less than the effort required to maintain a more complex solution. When the weightings change, we will re-evaluate our decision,
whether that means adopting a monorepo tool or splitting our codebase into multiple repositories.

## Conclusion

I hope this article has been helpful. In summary, adopting a monorepo requires consideration. Be practical rather than ideological.
Don't just follow the trend. Consider your own situation and make a decision that is right for you. Understand that
there is no one-size-fits-all solution and every solution has its own trade-offs (unless you're Google and you have the resources to build your own solution).
