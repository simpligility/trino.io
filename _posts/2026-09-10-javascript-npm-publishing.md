---
layout: post
title: "Trino JavaScript packages are back on npm"
author: "Manfred Moser"
excerpt_separator: <!--more-->
image: /assets/images/logos/javascript-small.png
---

Trino has an official JavaScript client and a new React query editor component.
For the better part of a year, neither project could publish a release to the
npm registry. Both are unblocked now. They publish automatically, under new
names, with trusted publishing and provenance attestations, and without a single
stored credential.

<!--more-->

## The two projects

In late 2024 I wrote about
[the many places JavaScript shows up in Trino]({{ site.baseurl }}{% post_url 2024-11-18-javascript %}),
from client drivers to the web user interfaces to the Grafana plugin. Two of
those threads matter here.

The [trino-js-client](https://github.com/trinodb/trino-js-client) project is the
official Trino client for Node.js, donated to the project by
[Filipe Regadas](https://github.com/regadas) and released for the first time
under the Trino umbrella in 2024. It is used by the
[Visual Studio Code support]({{ site.baseurl }}/ecosystem/client-application.html#vscode),
the [Emacs support]({{ site.baseurl }}/ecosystem/client-application.html#emacs),
and a long list of custom web applications.

The [trino-query-ui](https://github.com/trinodb/trino-query-ui) project is
newer. It packages a query editor as a reusable React component, with metadata
browsing, schema-aware SQL completion, multiple query tabs, and result set
inspection. It is meant to be embedded in the
[Trino Web UI]({{ site.baseurl }}/docs/current/admin/web-interface.html), and in
any other React application that needs a Trino query editor.

## What broke

In 2025 the npm registry was hit by a series of supply chain attacks that spread
through compromised publishing tokens. GitHub responded with a
[plan for a more secure npm supply chain](https://github.blog/security/supply-chain-security/our-plan-for-a-more-secure-npm-supply-chain/)
that phases out publishing with long-lived tokens entirely.

Both Trino projects published with a stored token, so both stopped publishing.
The last token-based releases were `trino-client` 0.2.9 in November 2025 and
`trino-query-ui` 0.1.1 in December 2025. The next release attempt built fine and
then failed at the publish step with `Access token expired or revoked`, which is
recorded in
[trino-query-ui issue 31](https://github.com/trinodb/trino-query-ui/issues/31).

That was more than an inconvenience. Without a published package, the query
editor could not be consumed as a dependency, and the work to bring it into the
Trino Web UI had nowhere to go. The issue collected a steady stream of people
asking when it would be fixed.

## Access first

The first obstacle was not technical. Configuring trusted publishing needs owner
access to the npm organization and admin access to the repositories, and working
out who held what took longer than the code changes did. Thanks to Martin
Traverso and Filipe Regadas for tracking down the accounts and sorting out the
access.

## Trusted publishing

With access in place, both repositories moved to
[npm trusted publishing](https://docs.npmjs.com/trusted-publishers). Instead of
reading an `NPM_TOKEN` secret, the release workflow authenticates with OpenID
Connect. npm verifies the identity of the workflow itself, down to the
repository and the workflow file name, and issues a short-lived credential for
that one run.

Nothing is stored in the repository, so there is no token to rotate, expire, or
leak. The trusted publisher configuration on npmjs.com is the single place that
decides which workflow may publish which package.

Trusted publishing also brings provenance. Every version published this way
carries a [SLSA provenance](https://slsa.dev/provenance/v1) attestation that
links the tarball on npm back to the commit and the workflow run that built it.
Consumers can check it:

```shell
npm audit signatures
```

## New package names

Both packages moved under the `@trinodb` scope on npm, which makes project
ownership of the packages explicit and gives future Trino JavaScript packages a
consistent name.

| Old name | New name | Current version |
|---|---|---|
| `trino-client` | `@trinodb/trino-js-client` | 0.3.1 |
| `trino-query-ui` | `@trinodb/trino-query-ui` | 0.1.5 |

The old names stay on npm at their last published version and receive no further
releases, so update the dependency name to keep getting new versions:

```shell
npm install @trinodb/trino-js-client
npm install @trinodb/trino-query-ui
```

## Releases are automated now

Both repositories use the same release flow. The workflow runs on every push to
`main` and checks whether the version in the package manifest changed. When it
did, the workflow publishes to npm and then creates the GitHub release with
generated notes. When it did not, the workflow does nothing.

Cutting a release is therefore a single reviewed pull request that bumps the
version. There is no manual tagging step and no manual publish.

The order of the two final steps matters. Publishing runs first, and the release
is created only after the publish succeeds. Doing it the other way around left
behind a GitHub release and a tag for a version that never reached npm, which
then had to be deleted by hand before the run could be repeated.

## Snags worth knowing

A few details cost real time, and they apply to any project making the same
move:

* **Trusted publishing cannot create a package that does not exist yet.** The
  registry has nothing to attach the trusted publisher configuration to. The
  first version under each scoped name has to be published by hand, and the
  workflow takes over from the next version onwards.
* **Use a current Node.js release.** The OIDC token exchange needs a recent npm
  CLI. Both repositories now build and release on Node.js 24.
* **Yarn behaves differently from npm.** The trino-js-client project builds with
  Yarn, which generates a provenance attestation only when it is asked to
  through `publishConfig`, and which otherwise publishes to a registry mirror
  rather than to the registry that holds the trusted publisher configuration.
  Both need an explicit setting. The move also required an upgrade from Yarn
  3.2.1 to 4.18.0.

Both repositories document the resulting release process in their readme files,
and the full trail of the work is captured in
[trino-query-ui issue 31](https://github.com/trinodb/trino-query-ui/issues/31).

## What's next

Publishing was the blocker, not the goal. With releases flowing again, the
following work becomes possible:

* Embed the query editor component in the Trino Web UI, which is the reason the
  trino-query-ui project exists.
  <!-- TODO: link the new tracking issue -->
* Take the query editor from its current early-stage state to something
  recommended for production use, with documentation to match.
  <!-- TODO: link the new tracking issue -->

The plans from the 2024 post are still on the list for the JavaScript client as
well, and several of them now have a way to reach users again:

* [Support authentication methods beyond basic authentication](https://github.com/trinodb/trino-js-client/issues/524)
* Add support for the spooling client protocol
* Improve the documentation and the example projects
* Test with Trino Gateway and adjust as needed

## Help wanted

All of this needs more hands. The JavaScript work in Trino is separate enough
from the query engine that you do not need to know Java, or Trino internals, to
be useful. If you write TypeScript, React, or GitHub Actions, there is something
here for you.

Have a look at the open issues in
[trino-js-client](https://github.com/trinodb/trino-js-client/issues) and
[trino-query-ui](https://github.com/trinodb/trino-query-ui/issues), come talk to
us in the `#core-dev` channel on [Trino Slack]({{ site.baseurl }}/slack.html),
and join an
[upcoming Trino contributor call]({{ site.baseurl }}/community.html#events).
