# Deploy Cloud Foundry locally using Bosh Lite

This captures my experience of deploying Cloud Foundry using bosh lite (v2) in my local MacBook.  
The steps are based on the following pages:
* Procedure to deploy bosh lite - https://bosh.io/docs/bosh-lite
* Procedure to deploy Cloud Foundry with Diego -
https://github.com/cloudfoundry/diego-release/tree/develop/examples/bosh-lite
* An excellent guide to deploy CF with an example app - https://mariash.github.io/learn-bosh/

# Deploy Bosh Lite
Follow this page to deploy bosh lite: https://bosh.io/docs/bosh-lite. This works pretty 
seamlessly in my experience.
 
# Deploy Cloud Foundry with Diego
The procedure below contains tweaks here from this page:
https://github.com/cloudfoundry/diego-release/tree/develop/examples/bosh-lite.  They are done to 
work with the v2 bosh CLI.

## Upload Stem Cells
The very first step is to upload stem cells.
```
bosh -e vbox upload-stemcell https://bosh.io/d/stemcells/bosh-warden-boshlite-ubuntu-trusty-go_agent
```

## Check out cf-release and diego-release
This is exactly the same as described in the original page:
```
cd ~/workspace
git clone https://github.com/cloudfoundry/cf-release.git
cd ~/workspace/cf-release
git checkout release-candidate # do not push to release-candidate
./scripts/update

cd ~/workspace
git clone https://github.com/cloudfoundry/diego-release.git
cd ~/workspace/diego-release
git checkout master # do not push to master
./scripts/update
```

## Generate manifests

### Install spiff
Install spiff according to its [README](https://github.com/cloudfoundry-incubator/spiff). spiff 
is a tool for generating BOSH manifests that is required in some of the scripts used below.

### Generate CF manifest
Instead of using the `generate-bosh-lite-dev-manifest` script as described in the original page, 
which does not work for bosh v2, please directly invoke `generate_deployment_manifest` as in the 
end of the `generate-bosh-lite-dev-manifest` script as follows:
```
mkdir -p ${CF_RELEASE_DIR}/bosh-lite/deployments
"${CF_RELEASE_DIR}/scripts/generate_deployment_manifest" \
  bosh-lite \
  <(echo "director_uuid: ${DIRECTOR_UUID}") \
  "$@" \
  > "${CF_RELEASE_DIR}/bosh-lite/deployments/cf.yml"
```
The `DIRECTOR_UUID` can be obtained using the following bosh v2 command, `bosh -e vbox env`, 
where `vbox` is the env alias when I deploy bosh lite.
 
### Generate Diego manifest
Prior to invoking the diego manifests generation script, we should skip generating the cf 
manifest again by setting the `CF_MANIFESTS_DIR` environment variable.
```
export CF_MANIFESTS_DIR=~/workspace/cf-release/bosh-lite/deployments
cd ~/workspace/diego-release
./scripts/generate-bosh-lite-manifests
```

## Upload and Deploy
The deployment commands below have been adjusted from the original commands to work with v2 bosh 
CLI.

### Deploy cf-release
```
cd ~/workspace/cf-release
bosh -e vbox upload-release &&
bosh -e vbox deploy -d cf-warden bosh-lite/deployments/cf.yml
```

### Upload cflinuxfs2 and garden-runc
```
bosh -e vbox upload-release https://bosh.io/d/github.com/cloudfoundry/cflinuxfs2-release
bosh -e vbox upload-release https://bosh.io/d/github.com/cloudfoundry/garden-runc-release
```

### Deploy diego-release
```
cd ~/workspace/diego-release
bosh -e vbox upload-release &&
bosh -e vbox deploy -d cf-warden-diego bosh-lite/deployments/diego.yml
```
