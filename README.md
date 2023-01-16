See [.yamato/build-and-upload.yml](../opentelemetry-cpp/.yamato/build-and-upload.yml) and the
[vcpkg yamato job](https://yamato.cds.internal.unity3d.com/jobs/1997-vcpkg)
for how this is used.

# Versioning scheme

The current Stevedore package version has three parts:

1. The official OpenTelemetry CPP version
2. The vcpkg baseline commit used to build the package
3. The vcpkg unity fork's `opentelemetry-cpp` branch commit used to build the package

`<opentelemetry-cpp_version>-<vcpkg_baseline_commit>-<unity_fork_commit>`

# Packaging a new opentelemetry-cpp version

First, check whether the version you want to package exists currently in [the known versions JSON file in the master branch](https://github.com/Unity-Technologies/vcpkg/blob/master/versions/o-/opentelemetry-cpp.json).

## The version exists in the JSON file

It should be a matter of specifying another version in the `upload to stevedore testing` job variables [when lauching through Yamato](https://unity-ci.cds.internal.unity3d.com/project/1997/branch/opentelemetry-cpp/jobDefinition/.yamato%2Fupload.yml%23upload_to_stevedore_testing)

## The version doesn't exist in the JSON file

1. Clone the repo locally
2. `git checkout master`
3. `git remote add upstream git@github.com:microsoft/vcpkg.git`
4. Look into [the official repo tags](https://github.com/microsoft/vcpkg/tags) and choose one (the latest, most probably)
5. `git pull --rebase upstream my-chosen-tag`
6. `git push origin master`
7. Take note of `master` latest commit's SHA (which should match the chosen tag above) using `git log`
8. `git checkout opentelemetry-cpp`
9. Create a new feature branch from the existing `opentelemetry-cpp` branch
10. Assign the `master` latest commit SHA you noted earlier to the `vcpkg_repo_baseline_commit` variable at the start of the [.yamato/build-and-upload.yml](../opentelemetry-cpp/.yamato/build-and-upload.yml) file in your feature branch
11. Assign the new `opentelemetry-cpp` version you want to the `opentelemetry_cpp_version` variable at the start of the [.yamato/build-and-upload.yml](../opentelemetry-cpp/.yamato/build-and-upload.yml) file in your feature branch
12. Once committed and pushed, test your changes through the following Yamato CI jobs on your feature branch:
    - `build linux`
    - `build mac x64`
    - `build mac arm64`
    - `build windows`
13. Once confirmed that the jobs are successful, open a PR from your feature branch to `opentelemetry-cpp` branch
14. Once merged, from [the opentelemetry-cpp branch in Yamato](https://unity-ci.cds.internal.unity3d.com/project/1997/branch/opentelemetry-cpp), run the `upload to stevedore testing` job

### A note about the "build mac arm64" job

As you might have noticed, we currently have to `./vcpkg install` grpc and copy the `x64` tools in the `arm64` build directory prior to `./vcpkg install` opentelemetry-cpp itself. See the `build mac arm64` job in the [.yamato/build-and-upload.yml](../opentelemetry-cpp/.yamato/build-and-upload.yml) file.

This is due to a new cross-compiling issue encountered in this job specifically: since we're cross-compiling (compiling `arm64` binaries on an `x64` machine), it seems at some point `vcpkg` gets confused while building  `opentelemetry-cpp` and searches for the `grpc_cpp_plugin` native executable in the `arm64` build directory, which it doesn't find (since we're on `x64`, `vcpkg` only builds the `x64` tools and put them in the `x64` build directory).

The job then passes `gRPC_CPP_PLUGIN_EXECUTABLE-NOTFOUND` to `opentelemetry-cpp` build commands, which fails.

Multiple fixes were tried other than the currently working `build grpc, copy tools, then build opentelemetry-cpp`, here are a couple notable ones:

1. Build both sets of tools (`arm64` and `x64`)
    - This didn't work as the `opentelemetry-cpp` build process still looks for tools in the `arm64` build dir, which it finds but can't execute (`bad CPU type in executable`)
2. Modify the `triplet` to include the path to the tools manually
    - The variables tried didn't work and `gRPC_CPP_PLUGIN_EXECUTABLE-NOTFOUND` was still being injected in the build commands
3. Add `x64` build directory in local `PATH` before running `./vcpkg install`
    - This didn't work, the build process doesn't seem to be looking for the tools in the `PATH` or the `PATH` variable didn't carry over
4. Modify [the opentelemetry-cpp portfile](https://github.com/Unity-Technologies/vcpkg/blob/f14984af3738e69f197bf0e647a8dca12de92996/ports/opentelemetry-cpp/portfile.cmake) to include the following code (taken from [the grpc portfile](https://github.com/Unity-Technologies/vcpkg/blob/f14984af3738e69f197bf0e647a8dca12de92996/ports/grpc/portfile.cmake#L24):
    - It didn't seem like `vcpkg` ran it even on a custom `baseline` commit in a custom branch. (Personal note from Émeric Morency) I also tried outputting messages in the `portfile`, but never saw them appear.
```
if(NOT TARGET_TRIPLET STREQUAL HOST_TRIPLET)
    vcpkg_add_to_path(PREPEND "${CURRENT_HOST_INSTALLED_DIR}/tools/grpc")
endif()
```

(Personal note from Émeric Morency) Maybe the last option could be a viable fix in the long run and someone with better `vcpkg` knowledge would be more successful than I was. I chose to timebox the trial & error or it wouldn't end and I don't have enough knowledge about `vcpkg` to try more complex solutions like adding a `git` patch for specific ports [like it's done in grpc, for example](https://github.com/microsoft/vcpkg/tree/f14984af3738e69f197bf0e647a8dca12de92996/ports/grpc).

#### If only the "build mac arm64" job fails for an opentelemetry-cpp version higher than 1.8.1

This probably means we also have to bump the `grpc` package version we're using for the `arm64` build (see the `macos_arm64_grpc_version` variable at the start of [.yamato/build-and-upload.yml](../opentelemetry-cpp/.yamato/build-and-upload.yml)).

In order to know which version we need, the most straightforward way is probably to:
1. Remove the following code from the `build mac arm64` job:
```
mv vcpkg.json vcpkg-otelcpp.json
mv vcpkg-grpc.json vcpkg.json
./vcpkg install
cp -r vcpkg_installed/x64-osx/tools/grpc/ vcpkg_installed/${VCPKG_DEFAULT_TRIPLET}/tools/grpc/
rm vcpkg.json
mv vcpkg-otelcpp.json vcpkg.json
```
2. Commit it in a feature branch and run the `build mac arm64` job in Yamato for your feature branch
3. At the start of the build (Execution logs), you should see which version of `grpc` your new `opentelemetry-cpp` build depends on:
```
The following packages will be built and installed:
  * abseil[core]:x64-osx -> 20220623.1 -- /Users/bokken/vcpkg/buildtrees/versioning_/versions/abseil/c569c0e44beca0b94d5a2d52a24e3a91868550ae
  * abseil[core]:arm64-osx-10.14-release -> 20220623.1 -- /Users/bokken/vcpkg/buildtrees/versioning_/versions/abseil/c569c0e44beca0b94d5a2d52a24e3a91868550ae
  * c-ares[core]:x64-osx -> 1.18.1#1 -- /Users/bokken/vcpkg/buildtrees/versioning_/versions/c-ares/15542c1c419b7874a8d3229cdf6366361e376a57
  * c-ares[core]:arm64-osx-10.14-release -> 1.18.1#1 -- /Users/bokken/vcpkg/buildtrees/versioning_/versions/c-ares/15542c1c419b7874a8d3229cdf6366361e376a57
  * grpc[codegen,core]:x64-osx -> 1.50.1 -- /Users/bokken/vcpkg/buildtrees/versioning_/versions/grpc/13420572b45b2b5a5a3fb41eb383d0c62f1d7c51
    grpc[core]:arm64-osx-10.14-release -> 1.50.1 -- /Users/bokken/vcpkg/buildtrees/versioning_/versions/grpc/13420572b45b2b5a5a3fb41eb383d0c62f1d7c51
[...]
```
4. Assign the new version number to the `macos_arm64_grpc_version` variable at the start of [.yamato/build-and-upload.yml](../opentelemetry-cpp/.yamato/build-and-upload.yml) alongside your other version changes.
