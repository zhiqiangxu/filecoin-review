# Debug builtin actor with snapshot

Here're some instructions you'll need if you want to just debug the builtin actor with downloaded snapshot.

```
# build builtin actor
cd path_to_built_actor
make bundle-mainnet

# update environment variables to point to builtin actor bundles
export LOTUS_FVM_DEVELOPER_DEBUG=1
# you may need to change V12 accordingly
export LOTUS_FVM_DEBUG_BUNDLE_V12=path_to_bundles

# make lotus client
cd path_to_lotus
make

# run lotus with snapshot, disabling bootstrap
./lotus daemon --bootstrap false --import-snapshot path_to_snapshot 
```