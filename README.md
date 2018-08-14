# CircleCI 2.0 bundle cache restore issue

A simple repository to demonstrate the CircleCI 2.0 bundle cache restore issue logged [here](https://discuss.circleci.com/t/unable-to-restore-bundle-from-cache-using-circleci-ruby-docker-images/24249). Below is just the text from that issue page:

---

Using the Docker image `circleci/ruby` it's impossible to restore a bundler cache saved to the default location of `/usr/local/bundle`.

The image already has a `/usr/local/bundle` directory pre-created by the `root` user.

```bash
$ ls -Al /usr/local
total 36
drwxrwsr-x 1 root staff 4096 Aug 11 00:06 bin
drwxrwsrwx 2 root staff 4096 Jul 21 08:36 bundle
drwxrwsr-x 1 root staff 4096 Jul 17 07:54 etc
drwxrwsr-x 2 root staff 4096 Jul 16 00:00 games
drwxrwsr-x 1 root staff 4096 Jul 21 08:36 include
drwxrwsr-x 1 root staff 4096 Jul 21 08:36 lib
lrwxrwxrwx 1 root staff    9 Jul 16 00:00 man -> share/man
drwxrwsr-x 2 root staff 4096 Jul 16 00:00 sbin
drwxrwsr-x 1 root staff 4096 Jul 17 03:16 share
drwxrwsr-x 2 root staff 4096 Jul 16 00:00 src
```

But the build tasks are run as the `circleci` user.

```bash
$ whoami
circleci
```

And the default `bundler` options are:

```bash
$ bundle config
Settings are listed in order of priority. The top value will be used.
app_config
Set via BUNDLE_APP_CONFIG: "/usr/local/bundle"

path
Set via BUNDLE_PATH: "/usr/local/bundle"

silence_root_warning
Set via BUNDLE_SILENCE_ROOT_WARNING: true
```

The `usr/local/bundle` directory has `drwxrwsrwx` permissions, so we can use a Circle 2.0 `config.yml` to `bundle install` and save the cache:

```yaml
  - run: bundle install
  - save_cache:
      key: v1-bundle-{{ checksum "Gemfile.lock" }}
      paths:
        - /usr/local/bundle
```

But when we come to restore the cache for the next job we get errors:

```bash
Found a cache from build 45 at v1-bundle-37RyclRo4zIg9I2R_L+I72Po+eXbphREAXxknkNduy8=
Size: 60 MB
Cached paths:
  * /usr/local/bundle

Downloading cache archive...
Validating cache...

Unarchiving cache...
tar: usr/local/bundle: Cannot utime: Operation not permitted
tar: usr/local/bundle: Cannot change mode to rwxr-sr-x: Operation not permitted
tar: Exiting with failure status due to previous errors
Error untarring cache: exit status 2
```

I think this is because of the problem described by Dave Hay in this [blog post](https://portal2portal.blogspot.com/2011/01/doh-error-tar-cannot-utime-operation.html). `tar` cannot set the file time for files in a folder that is owned by `root`.

```bash
$ mkdir ~/foo
$ chown root:root ~/foo
$ chmod a+w ~/foo
$ cd ~/foo
$ tar xvf /media/LenovoExternal/Products/DB297/db297linux32.tar
tar: .: Cannot utime: Operation not permitted
tar: Exiting with failure status due to previous errors
```

I cannot see a way round this with the `circleci/ruby` images. `restore_cache` *cannot* use `tar` in this folder.

My questions are:

1.  Why does this folder exist in the Docker image? It is empty.
2.  Why, if it must exist, can it not be owned by the `circleci` user?
3.  Is your implementation of `restore_cache` visible anywhere (on Github maybe), so I can check out what it's trying to do?
