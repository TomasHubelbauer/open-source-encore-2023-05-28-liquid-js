# Open Source Encore: LiquidJS shell script "portability" for macOS

In a previous installment of Open Source Encore on LiquidJS I have contributed
[the Jekyll `push` filter](https://github.com/harttle/liquidjs/pull/611),
[the Jekyll `sample` filter](https://github.com/harttle/liquidjs/pull/612) and
retroactively also tests for the `push` filter which I've neglected on the first
round: <https://github.com/harttle/liquidjs/pull/614>.

For the first PR (the `push`) one, I have not tested my changes, I figured if
the CI on the repository passes, I'll be good to go and I will just manually
test the change in the Liquid Playground.
It didn't work there.

The next time around, with the `sample` PR, I figured out how to run the tests
so I contributed that one with tests ready to go.

After that experience I figured I should go back and add `push` tests as well to
find out why `push` doesn't work on the Playground.
It was rather suspicious since it just used `concat` internally which was fully
tested but I knew how to add and run tests now so that's what I did.

Turns out the change was working perfectly fine, so I have yet to figure out why
the `push` filter won't work on the Playground.

But that's for another day.
This post is about my PR where I am retroactively adding the tests for `push`.
As a part of it, I added `CONTRIBUTING.md` and explained the steps to build and
run the changes and tests.

I neglected to document one thing, though.
The build command is actually two - one command to build the code and one to
build the docs.
For the tests to run your change, all you need to do is build the code.
But the command for building the docs was failing for me.
I pushed the problem down and soldiered on with the code changes and tests, but
every time I went to commit stuff, the pre-commit hook would re-surface the
issue because it is using the full build command.

At some point I got tired of bypassing the pre-commit hook checks so I delved
into researching what the problem could be.

Quite quickly I realized the issue is with a call to `sed` which is not a big
surprised because macOS is famous for shipping old versions of stuff and the
command lines between modern Unix based system like Ubuntu (used to run the
GitHub Actions workflow on the project) and the macOS utilities are rarely
portable.

I ran into this issue at first:

```
+ ./bin/build-contributors.sh
sed: -e: No such file or directory
sed: 1: "docs/themes/navy/layout ...": extra characters at the end of d command
```

This was caused by the `-i` flag used on the `sed` command:

```shell
sed -i \
    -e 's/README.md/docs\/themes\/navy\/layout\/partial\/all-contributors.swig/g' \
    -e 's/"contributorsPerLine": 7/"contributorsPerLine": 65535/g' \
    docs/.all-contributorsrc
```

You can see the `sed -i -e` sequence here, split across multiple lines.
Well, you can't do that on macOS!

Turns out you need to add `-i ''` to make the macOS `sed` like you.

<https://stackoverflow.com/a/62309999/2715716>

I fixed that by introducing the `''` and moved on.

The next issue was coming from these lines:

```shell
sed -i '1i ---\ntitle: Changelog\nauto: true\n---\n' source/tutorials/changelog.md
sed -i '1i ---\ntitle: 更新日志\nauto: true\n---\n' source/zh-cn/tutorials/changelog.md
```

I fixed them the same way but the issue wasn't going away.
Now I got this error:

```
command i expects \ followed by text
```

I looked at a few resources.
One seemed to suggest I needed to remove the space between the `-i` and `''` so
I did that to no avail.

<https://singhkays.com/blog/sed-error-i-expects-followed-by-text/>

I spent a lot of time on this, reading various articles and so on.
I don't have the original source handy, because it was buried deep within a long
session of just researching and trying everything, but one article prompted me
to consider the actual _command_, not the _flag_.

You see, in the above snippet, we do have the `-i` flag, yes, but the `sed`
_command_ is `1i ---\ntitle: Changelog\nauto: true\n---\n`.
The `i` here is the problem!

On macOS, for whatever reason, the command needs to look like this:

```shell
sed -i '' -e '1i\
 ---\ntitle: Changelog\nauto: true\n---\n' source/tutorials/changelog.md
```

That's `sed`, called with `-i ''` as we've already fixed, then followed by the
actual command string, which has the `1i` followed by the slash as the error
message wanted us to do, and then a newline and the original text that was
there.

The newline made all the difference.

At this point I gave up on trying to make a single portable command line and I
decided to do OS detection and conditionally call `sed` either the macOS/BSD
way to the GNU way.

I've again offered these changes: <https://github.com/harttle/liquidjs/pull/615>

Now new contributors get an honest contributing guide which suggests the full
build command (so the pre-commit hook doesn't fail on them) and also there
should be no issues for macOS contributors now.

We still have a problem with Windows contributors unless they use Cygwin or WSL,
but that can be solved down the line once it becomes a problem someone raises.

The solution should be to add a Batch / PowerShell script which would be run for
Windows users instead of these shell scripts, but that's something I can neither
test (as I don't have a Windows machine) nor feel like spending my time on, so
hopefully someone else will be motivated to.
