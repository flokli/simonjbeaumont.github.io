---
layout: post
title: Automating Github Pages and Coveralls.io from Travis
---

With the advent of [Travis-ci][1] it has never been easier
to setup continuous integration for your Github project. Just sign in to Travis
with your Github account, enable some repos, profit! It will then build all
commits that are pushed to your repo and alert you when you have broken the
build. You can even badge your repo with the current build status:

![build-badge](https://travis-ci.org/simonjbeaumont/ocaml-pci.svg?branch=master)

It even tests your pull-requests for you and badges them as passing or failing
so you know whether or not to merge:

![pull-request](/images/coverage-docgen/pullreq.png)

This is all great, but there's no reason your Travis job cannot be harnessed to
do some other _useful stuff!_ Here we'll show how use Travis to automatically:

1. Publish your docs to [_Github Pages_][2]; and
2. Upload your coverage metrics to [_Coveralls.io_][3].

## Publishing docs to Github Pages

### Building the docs using `ocamldoc`

The first step is to generate your docs at build time. For OCaml projects using
Oasis this is pretty easy and just involves adding a stanza to your `_oasis`
file like this example taken from the [`ocaml-pci`][4] library:

```
Document pci
  Type:                 ocamlbuild (0.4)
  BuildTools:           ocamldoc
  Title:                API reference for Pci
  XOCamlBuildPath:      .
  XOCamlBuildLibraries: pci
```

The usual `oasis setup` dance generates a `Makefile` with a target to build the
docs and so they can be simply generated by:

```sh
$ ./configure --enable-docs
$ make doc
```

### Publishing on Github Pages

Github Pages allows for any content pushed to the `gh-pages` branch of
a repository, `repo`, to be served at `https//user.github.io/repo`. This is


a very useful feature and can be used to publish documentation for the
repository in question. Here's an example of the documentation page hosted by
Github at [https//simonjbeaumont.github.io/ocaml-pci][5] which is just
displaying the static HTML that has been push to [`ocaml-pci#gh-pages`][6].

In order to push from Travis you'll need to generate an access token which can
be done from the _Personal access tokens_ section of your Github settings page:

![access-token-generation](/images/coverage-docgen/token.png)

Once you have this you can encrypt and upload it for use in the Travis build
environment:

```sh
$ gem install travis
$ travis encrypt -r user/repo GH_TOKEN=<token>
secure: "ABC123ABC123ABC123ABC123ABC123ABC123ABC123ABC123ABC123..."
```

This then needs adding to your `.travis.yml` in the `env:` section:

```YAML
env:
  global:
  - secure: "ABC123ABC123ABC123ABC123ABC123ABC123ABC123ABC123ABC123..."
```

This can be made easier if you execute `travis` from within your repo. This
way it can infer the repository and so eliminating the need for the `-r` option
and it can also add it to your `.travis.yml` for you with the `--add` flag:

```sh
$ travis encrypt GH_TOKEN=<token> --add
```

Now Travis has a secure token by which it can push to your repository you can
now instrument your build job to push for you. Travis sets up some [useful
environment variables][7] for you in the build VM which can be used to determine
whether or not to push the docs. For example, I didn't want to push the docs if
Travis was building a pull-request, only if it was building a commit so I made
use of `$TRAVIS_PULL_REQUEST`. Here's an [example][8]:


```sh
#!/bin/sh
set -e
set +x  # Make sure we're not echoing any sensitive data

./configure --enable-docs
make doc

if [ -z "$TRAVIS" -o "$TRAVIS_PULL_REQUEST" != "false" ]; then
  echo "This is not a push Travis-ci build, doing nothing..."
  exit 0
else
  echo "Updating docs on Github pages..."
fi

DOCDIR=.gh-pages
if [ -n "$KEEP" ]; then trap "rm -rf $DOCDIR" EXIT; fi
rm -rf $DOCDIR

git clone --quiet --branch=gh-pages https://${GH_TOKEN}@github.com/simonjbeaumont/ocaml-pci $DOCDIR > /dev/null

cp _build/pci.docdir/* $DOCDIR

git -C $DOCDIR config user.email "travis@travis-ci.org"
git -C $DOCDIR config user.name "Travis"
git -C $DOCDIR commit --allow-empty -am "Travis build $TRAVIS_BUILD_NUMBER pushed docs to gh-pages"
git -C $DOCDIR push origin gh-pages > /dev/null
```

That's it! You just need to get Travis to execute this script for you as part
of the job.

## Pushing coverage metrics to Coveralls

It's good to know how much your tests suck! For this it's handy to know just
how much of your program or library your tests actually exercise. Enter
<s>Yak!</s> *Bisect*!

### Toys you'll need in your toybox

[Bisect][9] is a code coverage tool for OCaml written by Xavier Clerc. It uses
the OCaml preprocessor to add instrumentation points using either `camlp4` or
the new `ppx` extension point mechanism. When linked against your project it
will spew out files from which it can generate reports in various forms using
`bisect-report` provided in the package.

[Ocveralls][10] is a tool that will take Bisect coverage data and produce JSON
output suitable for uploading to Coveralls. It also has some smarts to work out
what kind of CI it is running on and to send the coverage data directly to
Coveralls using the `--send` flag.

### Scripting it for Travis

One of the main things to work around is not wanting to have Bisect linked
against your project for normal compilation. If you do then it will run its
instrumentation code every time it is executed, even if your project is
a library consumed by other projects, linking in Bisect will result in this
data being produced _*every time your code is executed*_.

The workaround for this is not very elegant but it works. During the Travis job
you can use `sed` to add `bisect` (or `bisect_ppx`) as a build dependency of
your package as a one-off step:


```sh
...
pushd $COVERAGE_DIR
$(which cp) -r ../* .
...
sed -i '/BuildDepends:/ s/$/, bisect_ppx/' _oasis
oasis setup

./configure --enable-tests
make
...
```

Then you can run your test to generate the coverage data:

```sh
find . -name bisect* | xargs rm -f
./test_pci.native -runner sequential
```

Note the `-sequential` here. This is because I have used OUnit for my unit
tests which spawns off worker threads to execute the test cases. Bisect will
generate incomplete results in this case unless the `BisectThreads` package is
linked in.

Once the data has been generated you can use `ocveralls --send` to push this
data to Coveralls:

```sh
if [ -n "$TRAVIS" ]; then
  echo "\$TRAVIS set; running ocveralls and sending to coveralls.io..."
  ocveralls --prefix _build bisect*.out --send
fi
```

Done! This script has some extra padding but is [used here][11]. Coveralls can
then keep track of your coverage over build history:


![coverage-trend](/images/coverage-docgen/coveralls-trend.png)

It can also comment on your pull-request and become part of your gating
criteria for merging:

![coveralls-comment](/images/coverage-docgen/coveralls-comment.png)

And, of course... where would we be without a gratuitous badge for our repo's
REAMDE?!:

![coveralls-badge](https://coveralls.io/repos/simonjbeaumont/ocaml-pci/badge.svg?branch=master)

### Let the game begin

Now for the quest to keep that coverage creeping up!

[1]: https://travis-ci.org
[2]: https://pages.github.com
[3]: https://coveralls.io
[4]: https://github.com/simonjbeaumont/ocaml-pci/blob/9865f79/_oasis#L30-L35
[5]: https//simonjbeaumont.github.io/ocaml-pci
[6]: https://github.com/simonjbeaumont/ocaml-pci/tree/gh-pages
[7]: http://docs.travis-ci.com/user/environment-variables/#Default-Environment-Variables
[8]: https://github.com/simonjbeaumont/ocaml-pci/blob/9865f79/.docgen.sh
[9]: http://bisect.x9c.fr/
[10]: https://github.com/sagotch/ocveralls
[11]: https://github.com/simonjbeaumont/ocaml-pci/blob/9865f79/.coverage.sh