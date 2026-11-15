Building a provider
==================================

A provider adds something to Music Assistant: a music source, support for a kind of player, a
metadata source or a plugin. A provider can live in two places, and what each asks of you differs.

## Two places

### In the server repository: a built-in provider

A built-in provider ships with every Music Assistant server, and the team keeps it working through
every change to the server. Because the project supports it for years, a new built-in provider
asks for:

- demand you can show: users who want it, beyond yourself;
- a pull request on the user documentation that describes it;
- code that follows the code standards of the server and the usage policy, with tests;
- a named maintainer (you) who follows up on issues and on changes to the service, the device or
  the server.

You work in a fork of the server, and a maintainer reviews the new provider against the
[usage policy](https://github.com/music-assistant/.github/blob/main/USAGE_POLICY.md). After that,
you merge most changes to your own provider yourself through the review bot. A built-in provider
whose maintainer no longer follows up may be moved to the community catalog; that is announced
first, so someone can take it over. The
[contributing guide](https://github.com/music-assistant/.github/blob/main/CONTRIBUTING.md) and the
[developer guide of the server](https://github.com/music-assistant/server/blob/dev/DEVELOPMENT.md)
have the details, and the [front page](index.md) of this site sets up a development environment
for the server.

### In your own repository: a community provider

A community provider lives in your own repository, with your own documentation, support and
releases. Servers install it from the community catalog, a list the project keeps in the
[registry](https://github.com/music-assistant/community-catalog). Users see the community catalog
only after an administrator enables it and accepts that everything in it is installed at their
own risk.

The catalog asks less than the server repository, and the project supports nothing in it:

- a public repository with an open source licence;
- releases that pass the automated checks;
- a domain of its own: not one the server uses or reserves, and not one another community
  provider has;
- a provider that crosses none of the red lines below.

A person decides once, at admission, about what the provider does. The code is not reviewed, and
every later release reaches users without anyone looking at it.

## The red lines

The catalog declines a provider at admission, and blocks it when it is found later, for:

- music the user has no right to: getting around an account, a subscription tier or a region, or
  a source that offers music without the permission of its rights holders;
- getting around copy protection;
- downloading or exporting music to keep it (a cache used for playback is fine);
- harmful behaviour: sending the user's data anywhere but the service it connects to, loading
  code from somewhere else while it runs, or doing things its description does not mention;
- a provider that mainly exists to load other providers, or lists of sources from third parties,
  from outside the community catalog.

Allowed, at the user's own risk: a small audience, an unofficial API used with the user's own
account, reading a service's public web pages, free public content in the quality a service
offers without a subscription, and code that would not pass a review for the server. The
[community catalog rules](https://github.com/music-assistant/.github/blob/main/COMMUNITY_CATALOG_RULES.md)
say it in full.

A blocked provider is not loaded by any server; its users keep their settings and library. A
blocked version is skipped the same way.

## The template

The [community provider template](https://github.com/music-assistant/community-provider-template)
is where a community provider starts. It holds one small music source, a test that runs it in a
real server, a development container and CI. Its README takes you from **Use this template** to a
provider running in a server in a few minutes, with Docker and VS Code as the only requirements,
on Windows, macOS or Linux.

The template pins the server version you develop against, at a release tag or a commit, and
installs it from its source, which gives you type checking, completion in your editor and the test
fixture.

What a release has to look like:

- the provider is in a folder named after its domain;
- the release is a GitHub release whose tag is the version, such as `v1.2.0`;
- the requirements are package names with version specifiers: no URLs, markers or options;
- the manifest is plain JSON;
- the icon is an SVG of at most 5 KB that only draws; an icon that does not pass is left out.

The [registry's README](https://github.com/music-assistant/community-catalog#for-provider-maintainers)
lists the limits.

Once installed, a provider runs inside the user's server. It never writes into its own folder:
the server checks that folder against the release and replaces it when it differs. Settings,
cached values and files go through the server. Its requirements are installed next to the
server's own packages and never replace one; a provider whose requirements clash is not loaded.

## Where users get help

The `support_url` in the manifest is where you take questions and bug reports, such as the issues
of your repository. The **Get help** button of the app, on the settings page of the provider and in
its details in the Marketplace, opens it, and so does the link to its maintainers in the note the
app shows before it installs a provider from the community catalog. Without a `support_url`, the
server gives an installed community provider its repository as one, so questions reach you and not
the support repository of Music Assistant. The app only follows a link that starts with
`https://`.

## Testing

### In your repository

`uv run pytest` runs the tests of the template: quick checks of the manifest, the texts, the
requirements and the icon, and a test that boots a real server with your repository as its local
provider folder. The fixture comes with the server: `pytest_plugins = ["music_assistant.testing"]`
in `tests/conftest.py` gives a fixture `mass`, a started server in dev mode with your repository
as its local provider folder, kept off the network. It needs pytest-asyncio with
`asyncio_mode = "auto"`, and next to the server the requirements of its Sendspin provider
(`aiosendspin[server]` and `av`, at the versions in the server's
`music_assistant/providers/sendspin/manifest.json`), which the server starts at every boot. The
template has all of this in place.

`scripts/run.sh`, or the **Run Music Assistant** task in VS Code, starts a server in dev mode that
loads your provider, so you can try it in the app. Outside the container this is:

```sh
PYTHONDEVMODE=1 python -m music_assistant --dev-providers-dir /path/to/your/repository
```

Only a server in dev mode loads providers from a local folder.

### On your pull requests

CI runs the lint, the type check, the spelling check and your tests, and calls the registry's
reusable workflow
[`check-provider.yml`](https://github.com/music-assistant/community-catalog/blob/main/.github/workflows/check-provider.yml),
which runs the same checks the catalog runs on a release: the manifest, the requirements against
a server line, and loading the provider in the server image of that line.

## Joining the catalog

You ask for admission with a pull request on the registry that adds one file,
`providers/<domain>/provider.json`: where your provider lives and who maintains it. Whoever opens
it, and every maintainer it names, must own the repository or be a code owner in the manifest of
its latest release, and that release must pass the checks on at least one server line. The
[registry's README](https://github.com/music-assistant/community-catalog#admission) tells the rest.

## What the nightly test reports

The registry checks every new release, and every night it tests the newest versions of each listed
provider again against the current server of every server line it serves: the nightly, the beta
and the newest stable line.

- A version that passes for a line is offered to the servers of that line.
- A version that fails, or that stops loading on a line, is not offered there. Servers of that line
  that run it switch to the newest version that still passes there; when none does, the provider
  stops loading there.
- You get an issue on your repository with the reason and the output of the failing check, or on
  the registry when your repository has issues switched off.

With a beta that runs for several weeks, you usually know about a problem well before a stable
release reaches your users. To support the stable release and follow the nightly at the same time,
release two versions on different days: the registry runs every night and takes only the newest
release published since its last run. Each version is then offered where it passes.
