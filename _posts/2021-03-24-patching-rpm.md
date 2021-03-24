---
layout: single
classes: wide
title:  "Patching RPMs on Fedora"
date:   2021-03-24 00:00:00 -0600
categories: linux tricks
---

In the era of Dockerfiles and `npm`, RPM hacking is becoming a lost art.

I needed to patch `mutter`, the GNOME Window Manager, because its primary GPU selection code did not take into consideration eGPUs, causing slowness.
They've since provided a [mainline workaround](https://gitlab.gnome.org/GNOME/mutter/-/merge_requests/1562), but in the mean time I had to run a patch.
This is what I learned in doing so.

There's a few resources available on this subject already, but they're missing a few Fedora-specific shortcuts and info on scripting. I'll attempt to cover those gaps.

Note that my target platform is Fedora 33 on amd64.
This process should be fine for any RHEL-based distro.

The non-`dnf` commands are the same for SUSE, but you're on your own for what `zypper` commands are equivalent.

### Preparing the Patch
There's a lot of ways to do this.
My preferred way is directly againt the `git` repository of the target package.

Apply your changes, then use `git diff > foo.patch` to create a patch file.

I'll be using this patch file.
```diff
diff --git a/src/backends/native/meta-renderer-native.c b/src/backends/native/meta-renderer-native.c
index 2365152c7..32edb75cf 100644
--- a/src/backends/native/meta-renderer-native.c
+++ b/src/backends/native/meta-renderer-native.c
@@ -3768,6 +3768,17 @@ choose_primary_gpu_unchecked (MetaBackend        *backend,
           }
       }
 
+    /* Consider hardware-accelerated secondary GPUs with outputs (like eGPUs) first */
+    for (l = gpus; l; l = l->next)
+      {
+        MetaGpuKms *gpu_kms = META_GPU_KMS (l->data);
+
+        if (!meta_gpu_kms_is_boot_vga (gpu_kms) &&
+            meta_gpu_kms_can_have_outputs (gpu_kms) &&
+            gpu_kms_is_hardware_rendering (renderer_native, gpu_kms))
+          return gpu_kms;
+      }
+
     /* Prefer a platform device */
     for (l = gpus; l; l = l->next)
       {
```

### Patching
I highly recommend doing this inside a container.
Packages tend to pull in a boatload of build dependencies that you won't ever use again.

This command is for standard `podman`, but [Fedora Toolbox](https://docs.fedoraproject.org/en-US/fedora-silverblue/toolbox/) is a tool worth considering too.
```bash
podman run -it --rm -v $(pwd):/src:z fedora /bin/bash
```

Let's get some basic dependencies.
```bash
dnf install -y dnf-plugins-core rpm-build
```

Install build dependencies. Note that this tends to miss some basic dependencies that you haven't installed in the container, like `g++` and `make`.
```bash
dnf builddep -y mutter
dnf install -y g++
```

Download the SRPM.
```bash
dnf download --source mutter
```

Install the SRPM. This will extract the RPM sources to the `rpmbuild` folder in your home directory.
```bash
rpm ivh mutter*.rpm
```

Copy your patch to the `rpmbuild/SOURCES` directory.
Then, modify `rpmbuild/SPECS/mutter.spec` to refer to this patch.
You don't need to specify the absolute directory--`rpmbuild` automaticaly looks in the `SOURCES` directory.
```
<truncated>
License:       GPLv2+
#VCS:          git:git://git.gnome.org/mutter
URL:           http://www.gnome.org
Source0:       http://download.gnome.org/sources/%{name}/3.38/%{name}-%{version}.tar.xz
Patch0:        0001-window-actor-Special-case-shaped-Java-windows.patch
Patch1:        0001-Revert-build-Do-not-provide-built-sources-as-libmutt.patch

# This one right here!
Patch2: egpu.patch

BuildRequires: chrpath
<truncated>
```

Now fire away!
```bash
rpmbuild -ba rpmbuild/SPECS/mutter.spec
```

Marvel at the slowness of GCC.
Then, your prize will be in the `RPMS/x86_64` directory.

Copy it out of the container.
If you used the `podman` command above, the `/src` directory is where the host can be reached.

To install the new RPM to your host machine, use the `--force` directive, since we haven't updated the version number of this RPM.
```bash
sudo rpm -iv --force mutter*.rpm
```

You may also wish to version lock this package, to prevent it from being overwritten by updates.
```bash
sudo dnf install dnf-plugin-versionlock -y
sudo dnf versionlock add mutter
```

### Scripting
This script attempts to the above, automatically.

```bash
#!/bin/bash
set -euo pipefail

if [ -z "${1+x}" ] || [ -z "${2+x}" ]; then
	echo "usage: $0 [TARGET PACKAGE] [APPEND TO SUMMARY] [PATCH FILES]..."
	exit 1
fi

TARGET=$1
APPEND=$2
shift 2

install_deps() {
	sudo dnf install -y make rpm-build dnf-plugins-core
}

install_srpm() {
	dnf download --source "$TARGET" --destdir /tmp
	rpm -ivh "/tmp/$TARGET*"
}

cp_patches() {
	for PATCH in "$@"; do
		cp "$PATCH" "$HOME/rpmbuild/SOURCES/"
	done
}

fix_spec() {
	SPECFILE="$HOME/rpmbuild/SPECS/$TARGET.spec"
	LATEST_PATCH=$(grep -o 'Patch[0-9]\+:' "$SPECFILE" | tail -n1 || true)
	if [ -z "$LATEST_PATCH" ]; then
		PATCHNUMBER=0
	else
		PATCHNUMBER=$(($(echo "$LATEST_PATCH" | grep -o '[0-9]\+') + 1))
	fi

	for PATCH in "$@"; do
		FULLPATCH="Patch$PATCHNUMBER: $PATCH"
		if [ -z "$LATEST_PATCH" ]; then
			sed -i "/^Source0:.*/a $FULLPATCH" "$SPECFILE"
		else
			sed -i "/^$LATEST_PATCH.*/a $FULLPATCH" "$SPECFILE"
		fi

		PATCHNUMBER=$((PATCHNUMBER + 1))
		LATEST_PATCH=$FULLPATCH
	done

	sed -i "/Summary:/ s/$/ $APPEND/" "$SPECFILE"
}

build() {
	sudo dnf builddep "$TARGET" -y
	rpmbuild -ba "$HOME/rpmbuild/SPECS/$TARGET.spec"
}

install_deps
install_srpm
cp_patches "$@"
fix_spec "$@"
build
```

Note that you'll have to run this in a container yourself.
It also might not cover all the needed dependencies.
Install them before running.

Example usage: 
```bash
patch-rpm.sh mutter "patched for eGPU usage" egpu.patch
```

### Summary
Patching an RPM is not terribly difficult.
Doing it in a container is convenient and keeps you from polluting your system.
There's also handy shortcuts, like `dnf builddep` and `dnf download --source` that make this quick and easy.
