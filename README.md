# RHEL Image mode - Openshift Pipelines CI/CD

This repository contains tasks and pipelines to implement a simple RHEL Image Mode building and converting workflow

It comes with a configuration playbook and vars to set-up everything in Red Hat Ansible Automation Platform (projects, credentials, etc) to start working with it.

## Requirements

### ServiceAccount privileges for the tasks
For the pipelines to properly work, the *podman* tasks (image building and bootc-image-builder) require **privileged** execution access.

One solution could be to add the *privileged SCC* to the **pipeline** service account:

```bash
oc adm policy add-scc-to-user privileged -z pipeline -n <YOUR_PROJECT>
```

For the VM Deployment
### Container registry credentials

All the tasks require credentials to fetch container images from **registry.redhat.io** and to fetch/push images to a destination container registry, for instance **quay.io**.
You can store the credentials in a secret that can be then associated to the workspaces in the pipelines:

```bash
podman login registryA
podman login registryB
oc create secret generic registry-credentials --from-file=.dockerconfigjson=$XDG_RUNTIME_DIR/containers/auth.json --type=kubernetes.io/dockerconfigjson -n <YOUR_PROJECT>
```

### RHEL Entitlements for RHEL Image mode builds

To build RHEL Image Mode container images, entitlements for RHEL are needed.
Entitlements can be fetched directly from the **openshift-config-managed** project:

```bash
oc get secret etc-pki-entitlement -n openshift-config-managed -o json | \
  jq 'del(.metadata.resourceVersion)' | jq 'del(.metadata.creationTimestamp)' | \
  jq 'del(.metadata.uid)' | jq 'del(.metadata.namespace)' | \
  oc -n <YOUR_PROJECT> create -f -
```

It will create a secret called **etc-pki-entitlement** in YOUR_PROJECT that can be then bound to the pipeline.

## Configuring the tasks

The repo comes with two different tasks:

- task-image-mode-build.yml -> To handle the creation and push of the container image
- task-bootc-image-builder.yml -> To handle the conversion of the container image to a specific format (QCOW2, ISO, AMI, VMDK)

Apply the tasks to the cluster:

```bash
oc apply -f task-bootc-image-builder.yml -f task-image-mode-build.yml -n <YOUR_PROJECT>
```

### Parameters for task-bootc-image-builder.yml

| Parameter              | Description                                      | Required | Default Value |
|------------------------|--------------------------------------------------|----------|--------------|
| `SOURCE_IMAGE`        | Reference of the source RHEL bootc image         | Yes      | `$(params.SOURCE_IMAGE_NAME)` |
| `SOURCE_IMAGE_TAG`    | Reference of the image tags Podman will produce | Yes      | `$(params.SOURCE_IMAGE_TAG)` |
| `BUILDER_IMAGE`       | Location of the Podman builder image            | Yes       | `registry.redhat.io/rhel9/bootc-image-builder:latest` |
| `DEST_FORMAT`         | Output format for the image                      | Yes       | `qcow2` |
| `CONFIG_TOML_CONTENT` | `config.toml` content for customizations         | No       | ```[[customizations.user]] name = "sysadmin" password = "redhat" groups = ["wheel"]``` |
| `TLS_VERIFY`         | Enable TLS verification                           | No       | `true` |
| `AWS_AMI_NAME`       | AWS AMI Name                                      | No       | `""` (empty) |
| `AWS_S3_BUCKET`      | AWS S3 Bucket                                     | No       | `""` (empty) |
| `AWS_S3_REGION`      | AWS S3 Region                                     | No       | `""` (empty) |


### Parameters for task-image-mode-build.yml

| Parameter         | Description                                                          | Required | Default Value |
|------------------|----------------------------------------------------------------------|----------|--------------|
| `IMAGE`         | Reference of the image Podman will produce                          | Yes      | `$(params.IMAGE_NAME)` |
| `IMAGE_TAGS`    | Reference of the image tags Podman will produce                     | Yes      | `$(params.IMAGE_TAGS)` |
| `BUILDER_IMAGE` | Location of the Podman builder image                                | Yes       | `registry.redhat.io/rhel9/podman` |
| `DOCKERFILE`    | Path to the Containerfile to build                                   | Yes       | `./Containerfile` |
| `CONTEXT`       | Path to the directory to use as context                             | es       | `.` |
| `TLSVERIFY`     | Verify TLS on the registry endpoint (for push/pull to a non-TLS registry) | No       | `true` |
| `FORMAT`        | Format of the built container (`oci` or `docker`)                    | No       | `oci` |
| `BUILD_EXTRA_ARGS` | Extra parameters passed for the build command                     | No       | `""` (empty) |
| `PUSH_EXTRA_ARGS`  | Extra parameters passed for the push command                      | No       | `""` (empty) |
| `SKIP_PUSH`     | Skip pushing the built image                                        | No       | `false` |

### Parameters for task-kubevirt-vm.yml

| Parameter         | Description                                                          | Required | Default Value |
|------------------|----------------------------------------------------------------------|----------|--------------|
| `SOURCE_IMAGE_NAME`        | Reference of the source RHEL bootc image         | Yes      | `N/A` |
| `SOURCE_IMAGE_TAG`    | Reference of the image tags Podman will produce | Yes      | `N/A` |
| `VM_NAME` | Name for the VM                               | Yes       | `N/A` |
| `VM_CPU`    | CPU Count for the VM                                   | Yes       | `2` |
| `VM_MEMORY`       | Memory count in Mi/Gi for the VM                          | Yes       | `2Gi` |
| `VM_DISK_SIZE`       | Disk Size in Mi/Gi for the VM                          | Yes       | `20Gi` |



## Configuring the pipelines

The repo comes with two pipeline:

- pipeline-image-build.yml -> Combines a Git clone task to fetch a Containerfile from a repository with the *task-image-mode-build* task we defined before
- pipeline-bootc-image-builder.yml -> Uses **bootc-image-builder** to convert a RHEL Bootc container image to a preferred format.
- pipeline-deploy-vm.yml -> It generates a ContainerDisk for Kubevirt/OCP Virtualization with a QCOW2 file and spins a VM.

Each pipeline has its own workspaces to manage Registry Credentials,

Apply the tasks to the cluster:

```bash
oc apply -f pipeline-image-build.yml -f pipeline-bootc-image-builder.yml -f pipeline-deploy-vm.yml -n <YOUR_PROJECT>
```

### Details for pipeline-image-build.yml

**Parameters**

| Parameter               | Description                                               | Required | Default Value            |
|-------------------------|-----------------------------------------------------------|----------|--------------------------|
| `GIT_REPOSITORY`       | URL of the repository where the Containerfile is stored  | Yes      | N/A                      |
| `GIT_REPOSITORY_BRANCH` | Branch of the repository to use                         | Yes       | `main`                   |
| `CONTAINERFILE_PATH`    | Path to the Containerfile, relative                     | Yes       | `./Containerfile`        |
| `BUILD_CONTEXT`        | Podman build context                                    | Yes       | `.`                      |
| `IMAGE_NAME`          | Name of the image to create (e.g., quay.io/user/image)  | Yes      | N/A                      |
| `IMAGE_TAGS`          | Tags to apply to the image, separated by a space        | Yes      | N/A                      |

**Workspaces**

| Workspace         | Description                                                                                      | Required |
|------------------|--------------------------------------------------------------------------------------------------|----------|
| `main-workspace`    | Stores the cloned Git repository containing the Containerfile and context for building the image | Yes      |
| `registry-creds`    | Used to provide authentication credentials for Podman to access container registries             | No       |
| `rhel-entitlements` | Stores Red Hat entitlement keys, allowing the build process to access subscription-based content | No       |


### Details for pipeline-bootc-image-builder.yml


**Parameters**

| Name                     | Description                                                               | Required | Default Value                          |
|--------------------------|---------------------------------------------------------------------------|----------|----------------------------------------|
| `SOURCE_IMAGE_NAME`      | Image to convert - e.g., `quay.io/kubealex/rhel-bootc-demo`             | No       | `quay.io/kubealex/rhel-bootc-demo`   |
| `SOURCE_IMAGE_TAG`       | Image tag                                                               | No       | `latest`                              |
| `DESTINATION_IMAGE_FORMAT` | Resulting image format - allowed: `qcow2`, `anaconda-iso`, `vmdk`, `raw`, `ami`, `gcp` | No       | `qcow2`                               |
| `CONFIG_TOML_CONTENT`    | `config.toml` content for customizations                               | No       | (Default user customization config)   |
| `TLS_VERIFY`             | TLS verification                                                       | No       | `true`                                |
| `AWS_AMI_NAME`           | AWS AMI Name (Only needed for AMI output)                                                          | No       | (empty)                               |
| `AWS_S3_BUCKET`          | AWS S3 Bucket (Only needed for AMI output)                                                          | No       | (empty)                               |
| `AWS_S3_REGION`          | AWS S3 Region (Only needed for AMI output)                                                          | No       | (empty)                               |
| `KUBEVIRT_IMAGE_BUILD`          | When QCOW2 format is chosen, option to create a ContainerDisk for the Image                                                  | No       | false                              |

**Workspaces**

| Workspace        | Description                                                                        | Required |
|-----------------|------------------------------------------------------------------------------------|----------|
| `main-workspace`          | Main workspace where the source files are stored                                  | Yes      |
| `registry-creds` | Used to provide authentication credentials for accessing container registries     | No       |
| `aws-creds`     | Stores AWS credentials for pushing images to AWS S3 or registering AMIs          | No       |

### Details for pipeline-deploy-vm.yml


**Parameters**

| Name                     | Description                                                               | Required | Default Value                          |
|--------------------------|---------------------------------------------------------------------------|----------|----------------------------------------|
| `SOURCE_IMAGE_NAME`      | Image to use for the VM - e.g., `quay.io/kubealex/rhel-bootc-demo`             | No       | `quay.io/kubealex/rhel-bootc-demo`   |
| `SOURCE_IMAGE_TAG`       | Image tag                                                               | No       | `kubevirt-qcow2`                              |
| `VM_NAME` | Name for the VM                               | Yes       | `N/A` |
| `VM_CPU`    | CPU Count for the VM                                   | Yes       | `2` |
| `VM_MEMORY`       | Memory count in Mi/Gi for the VM                          | Yes       | `2Gi` |
| `VM_DISK_SIZE`       | Disk Size in Mi/Gi for the VM                          | Yes       | `20Gi` |


**Workspaces**

N/A
