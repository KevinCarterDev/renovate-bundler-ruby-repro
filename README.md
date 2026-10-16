# Renovate: Bundler runs on the previous branch's Ruby

Minimal reproduction for a regression introduced in Renovate 44.90.0 by
[renovatebot/renovate#46084](https://github.com/renovatebot/renovate/pull/46084)
(memoise which `install-tool`s we've run).

## Setup

- `Gemfile` pins `ruby "4.0.6"` and depends on `rake`, locked at 13.3.1.
- `Gemfile.lock` was generated with Ruby 4.0.6 and Bundler 4.0.19.
- `renovate.json` gives `rake` a higher `prPriority`, so its branch is
  processed before the Ruby update in the same run.

## Expected

On the `ruby` update branch, the bundler artifact step runs `bundler lock`
under Ruby 4.0.7, and `Gemfile.lock` gets:

```diff
 RUBY VERSION
-  ruby 4.0.6
+  ruby 4.0.7
```

## Actual

`Gemfile.lock` is left at `ruby 4.0.6`, so `bundle install` in frozen mode
fails on the branch:

```
Bundler is unlocking ruby, but the lockfile can't be updated because frozen mode is set
```

## Cause

1. The `rake` branch runs `install-tool ruby 4.0.6` and
   `install-tool bundler 4.0.19`. containerbase installs Bundler under
   `/opt/containerbase/tools/bundler/4.0.19/4.0.6/`, and the generated
   binstub's shebang is `#!/opt/containerbase/tools/ruby/4.0.6/bin/ruby`.
2. The `ruby` branch runs `install-tool ruby 4.0.7`, but skips
   `install-tool bundler 4.0.19` because Bundler 4.0.19 is already memoised
   as active.
3. The `bundler` wrapper still points at the copy installed for Ruby 4.0.6,
   so `bundler lock` runs on Ruby 4.0.6 and writes that as the locked Ruby.

The containerbase half can be seen without Renovate:

```sh
docker run --rm --entrypoint bash ghcr.io/renovatebot/renovate:44.149.2 -c '
install-tool ruby 4.0.6 >/dev/null 2>&1
install-tool bundler 4.0.19 >/dev/null 2>&1
install-tool ruby 4.0.7 >/dev/null 2>&1
ruby --version
mkdir -p /tmp/t && cd /tmp/t
printf "source \"https://rubygems.org\"\nruby \"4.0.7\"\n" > Gemfile
bundler lock >/dev/null 2>&1
grep -A1 "RUBY VERSION" Gemfile.lock
'
```

This prints `ruby 4.0.7` for `ruby --version`, then `ruby 4.0.6` in the
lockfile.

## Reproducing with Renovate

```sh
docker run --rm \
  -e RENOVATE_PLATFORM=github \
  -e RENOVATE_TOKEN \
  -e RENOVATE_REPOSITORIES=<owner>/<fork> \
  -e LOG_LEVEL=debug \
  ghcr.io/renovatebot/renovate:44.149.2
```
