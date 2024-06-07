# Android 10 Ruby Termux Demo

This is a playground for patching ruby's `require_relative` method to fix running the `gem` binary in the ruby bootstrap package on Android 10 and above with Termux.

## Context

> The issue is originally described at [Termux#20359](https://github.com/termux/termux-packages/issues/20359).

Termux's build system let's you create bootstrap binaries for Android 10 as a workaround to the R^W violation (execution of externally loaded binary files not allowed).

If you try to bootstrap [`ruby`](https://github.com/termux/termux-packages/tree/master/packages/ruby) with the `--android10` flag and run it through the Termux app on the [`android-10`](https://github.com/termux/termux-app/tree/android-10) branch, the original binary works but the `gem` binary fails with the following error:

```shell
1|:/data/data/sh.gourav.jekyllex/files/home $ ruby --version
ruby 3.2.2 (2023-03-30 revision e51014f9c0) [aarch64-linux-android]
1|:/data/data/sh.gourav.jekyllex/files/home $ gem -v
/data/data/sh.gourav.jekyllex/files/usr/lib/ruby/3.2.0/rubygems.rb:15:in `require_relative': cannot load such file -- /data/app/~~MgtAZLOFHbuuc198BXGOIg==/sh.gourav.jekyllex-vGlBDbZNe0Nx_qSZi9YAkA==/lib/arm64/rubygems/compatibility (LoadError)
        from /data/data/sh.gourav.jekyllex/files/usr/lib/ruby/3.2.0/rubygems.rb:15:in `<top (required)>'
        from <internal:gem_prelude>:2:in `require'
        from <internal:gem_prelude>:2:in `<internal:gem_prelude>'
1|:/data/data/sh.gourav.jekyllex/files/home $
```

The steps to reproduce this are as follows:

```shell
git clone https://github.com/termux/termux-packages --depth 1
cd termux-packages
./scripts/run-docker.sh ./clean.sh
./scripts/run-docker.sh ./scripts/build-bootstraps.sh --android10
```

where `build-bootstrap.sh` is [modified](https://github.com/gouravkhunger/android10-ruby-termux-demo/blob/original-android10-ruby-build/patches/termux-packages/build-bootstraps.sh.patch) to build just `ruby`:

```shell
PACKAGES+=("ruby")
```

Through the [`build-bootstraps.yml`](https://github.com/gouravkhunger/android10-ruby-termux-demo/blob/main/.github/workflows/build-bootstraps.yml) workflow, this faulty bootstrap is released for testing under [Original android 10 ruby build](https://github.com/gouravkhunger/android10-ruby-termux-demo/releases/tag/bootstrap-2024.06.07-r1).

The `termux-app` is [patched](https://github.com/gouravkhunger/android10-ruby-termux-demo/tree/original-android10-ruby-build/patches/termux-app) to load this bootstrap and built by the [build-apks.yml](https://github.com/gouravkhunger/android10-ruby-termux-demo/blob/main/.github/workflows/build-apk.yml) workflow to provide a [debug apk](https://github.com/gouravkhunger/android10-ruby-termux-demo/releases/download/bootstrap-2024.06.07-r1/app-debug.apk) which can be quickly installed to verify this issue.

## Reason

In ruby `require_relative` assumes the file to be loaded is relative to the current file. 

In `/data/data/<package>/files/usr/lib/ruby/3.2.0/rubygems.rb`, there are `require_relative` statements:

```ruby
# ...
require_relative "rubygems/compatibility"

require_relative "rubygems/defaults"
require_relative "rubygems/deprecate"
# ...
```

It expects that the `compatibility.rb`, `defaults.rb` file must be in `/data/data/<package>/files/usr/lib/ruby/3.2.0/rubygems/*`, which in fact are there when you inspect the bootstrap and the device file explorer.

But in android 10, `gem` is not able to find them as the binary is loaded from a dynamic path which was `/data/app/~~MgtAZLOFHbuuc198BXGOIg==/<sh.gourav.jekyllex>-vGlBDbZNe0Nx_qSZi9YAkA==/lib/arm64/` in the original case, but the required files weren't able to be loaded relatively.

> realpath is apparently used to prevent double loading of files [[source]](https://github.com/termux/termux-packages/issues/20359#issuecomment-2145247719)

Which means that symbolic links are resolved before loading the file. This is the reason why the `require_relative` method fails to load the `rubygems/compatibility` file.

Android 10 and above do not allow executing files from the app's writable home directory unless they have the correct context type, and those too should be loaded as native libraries in the APK. Since termux sets up symlinks to the shared libraries from the bootstrap, the `require_relative` method fails to load the files because of the symlink resolution.

## Solution

In `v3.2.2`, which is the one Termux currently hosts, ruby's `require_relative` is internally handled by the procedure [`rb_f_require_relative`](https://github.com/ruby/ruby/blob/e51014f9c05aa65cbf203442d37fef7c12390015/load.c#L1479)

```c
VALUE
rb_f_require_relative(VALUE obj, VALUE fname)
{
    VALUE base = rb_current_realfilepath();
                 ^^^^^^^^^^^^^^^^^^^^^^^^^
    if (NIL_P(base)) {
        rb_loaderror("cannot infer basepath");
    }
    base = rb_file_dirname(base);
    return rb_require_string(rb_file_absolute_path(fname, base));
}
```

Here, `rb_current_realfilepath` is used to resolve the base path of the current file. This method is defined in [`vm_eval.c`](https://github.com/ruby/ruby/blob/e51014f9c05aa65cbf203442d37fef7c12390015/vm_eval.c#L2551-L2570):

```c
VALUE
rb_current_realfilepath(void)
{
    const rb_execution_context_t *ec = GET_EC();
    rb_control_frame_t *cfp = ec->cfp;
    cfp = vm_get_ruby_level_caller_cfp(ec, RUBY_VM_PREVIOUS_CONTROL_FRAME(cfp));
    if (cfp != NULL) {
        VALUE path = rb_iseq_realpath(cfp->iseq);
        if (RTEST(path)) return path;
        // eval context
        path = rb_iseq_path(cfp->iseq);
        if (path == eval_default_path) {
            return Qnil;
        }
        else {
            return path;
        }
    }
    return Qnil;
}
```

Here, ruby first tries to resolve the symlinks of the current instruction sequence (`rb_iseq_realpath`) and if possible returns it. If the path is not resolved, it tries to get the path without symlink resolution (`rb_iseq_path`). The two mentioned methods reside in [`iseq.c`](https://github.com/ruby/ruby/blob/e51014f9c05aa65cbf203442d37fef7c12390015/iseq.c) but the interesting thing is they differ in their implementation just regarding the symlink resolution part.


Here's our solution! We can [patch `vm_eval.c`](https://github.com/gouravkhunger/android10-ruby-termux-demo/blob/patched-android10-ruby-build/patches/termux-packages/packages/ruby/require_relative.patch) to use just the `rb_iseq_path` variant instead of `rb_iseq_realpath`.

```patch
diff --git a/vm_eval.c b/vm_eval.c
index 2e1a9b80a6..f2409e1e9e 100644
--- a/vm_eval.c
+++ b/vm_eval.c
@@ -2555,7 +2555,7 @@ rb_current_realfilepath(void)
     rb_control_frame_t *cfp = ec->cfp;
     cfp = vm_get_ruby_level_caller_cfp(ec, RUBY_VM_PREVIOUS_CONTROL_FRAME(cfp));
     if (cfp != NULL) {
-        VALUE path = rb_iseq_realpath(cfp->iseq);
+        VALUE path = rb_iseq_path(cfp->iseq);
         if (RTEST(path)) return path;
         // eval context
         path = rb_iseq_path(cfp->iseq);
```

## Conclusion

The fixed faulty bootstrap is released for testing under [Patched android 10 ruby build](https://github.com/gouravkhunger/android10-ruby-termux-demo/releases/tag/bootstrap-2024.06.07-r1). The release has an [`app-debug.apk`](https://github.com/gouravkhunger/android10-ruby-termux-demo/releases/download/bootstrap-2024.06.07-r2/app-debug.apk) which can be quickly installed to verify resolution.

```shell
:/data/data/sh.gourav.jekyllex/files/home $ ruby -v
ruby 3.2.2 (2023-03-30 revision e51014f9c0) [aarch64-linux-android]
:/data/data/sh.gourav.jekyllex/files/home $ gem -v
3.4.10
```

Run `gem update --system v3.5.11` and restart the app to see the updated version!
