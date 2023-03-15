# Storage Validation Tool for IBM Cloud Paks

Kubernetes has gained a lot of momentum with storage vendors providing support on various container orchestration platforms with [CSI](https://kubernetes-csi.github.io/docs/drivers.html) drivers and other mechanisms.

It has become essential for platform administrators to quickly validate a storage platform for their modernized workloads on IBM Cloud Paks and check its readiness level.

This Ansible Playbook helps functionally validate a storage on `ReadWriteOnce` and `ReadWriteMany` volumes. Note that these tests covers readiness and are only meant to be a pre-cursor to a full blown test with actual Cloud Pak workloads.

The following tests are performed:

 - Dynamic provisioning of a volume
 - Mounting volume from a node
 - Sequential [Read Write Consistency](./roles/storage-readiness/README.md#read-write-tests) from single and multiple nodes
 - Parallel Read Write Consistency from single and multiple nodes
 - Parallel Read Write Consistency across multiple threads
 - File Permissions on mounted volumes
 - Accessibility based on POSIX compliance [Group ID Permissions](./roles/storage-readiness/README.md#gid-tests)
 - [SubPath](https://kubernetes.io/docs/concepts/storage/volumes/#using-subpath) test for volumes
 - [File Locking](https://pubs.opengroup.org/onlinepubs/9699919799/functions/fcntl.html) test

### Prerequisites

- Ensure you have python 3.6 or later and [pip](https://pip.pypa.io/en/stable/installation/) 21.1.3 or later installed. We strongly
recommend to use Pyhton 3.8 or 3.9 or later as 3.6 will cause some feature deprecation warning on its removal in newer Ansible versions.

Examples below will be performed using Python 3.8 (adjust the commands to your Python versions accordingly, e.g. replace 3.8 in the commands below with 3.9 for Python 3.9)

```
# install Python 3.8
dnf install python38

# confirm the Python version
python3.8 --version

# check existinig pip version
python3.8 -m pip --version

# upgrade pip to the latest version
python3.8 -m pip install --upgrade pip
```

- Install Ansible 2.10.5 or later

```
# install the latest Ansible version
python3.8 -m pip install ansible
```

- Install ansible k8s modules

  `python3.8 -m pip install openshift`

  `ansible-galaxy collection install operator_sdk.util`

  `ansible-galaxy collection install community.kubernetes`

- Install [OpenShift Client 4.6 or later](https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.6.31) based on your OS.

- Access to the OpenShift Cluster (at least 3 compute nodes) setup with RWX and RWO storage classes with cluster admin access.

### Setup

 - Clone this git repo to your client

 - Update the `params.yml` file with your OCP URL and Credentials

   ```
    ocp_url: https://<required>:6443
    ocp_username: <required>
    ocp_password: <required>
    ocp_token: <required if user/password not available>
   ```
   
where you either modify the `ocp_username` and `ocp_password` entries OR just the `ocp_token`. Leave the entries that are not used intact.


 - Update the `params.yml` file for the `required` storage parameters

    ```
    storageClass_ReadWriteOnce: <required>
    storageClass_ReadWriteMany: <required>
    storage_validation_namespace: <required>
    ```

### Running the Playbook

 - From the root of this repository, run:

  ```bash
    ansible-playbook main.yml --extra-vars "@./params.yml" | tee output.log
  ```

  If the playbook fails to run due to SSL verification error, you can disable it by setting this environment variable before running the playbook

  ```
  export K8S_AUTH_VERIFY_SSL=no
  ```


### Running the Playbook with the Container

#### Environment Setup

```sh
export dockerexe=podman # or docker
export container_name=k8s-storage-test
export docker_image=icr.io/cpopen/cpd/k8s-storage-test:v1.0.0

alias k8s_storage_test_exec="${dockerexe} exec ${container_name}"
alias run_k8s_storage_test="k8s_storage_test_exec ansible-playbook main.yml --extra-vars \"@/tmp/work-dir/params.yml\" | tee output.log"
alias run_k8s_storage_test_cleanup="k8s_storage_test_exec cleanup.sh -n ${NAMESPACE} -d"
```

#### Start the Container

```sh
mkdir -p /tmp/k8s_storage_test/work-dir
cp ./params.yml /tmp/k8s_storage_test/work-dir/params.yml

${dockerexe} pull ${docker_image}
${dockerexe} run --name ${container_name} -d -v /tmp/k8s_storage_test/work-dir:/tmp/work-dir ${docker_image}
```

#### Run the Playbook

```sh
run_k8s_storage_test
```

#### Optional Cleanup the Cluster

```sh
run_k8s_storage_test_cleanup

[INFO ] running clean up for namespace storage-validation-1 and the namespace will be deleted
[INFO ] please run the following command in a terminal that has access to the cluster to clean up after the ansible playbooks

oc get job -n storage-validation-1 -o name | xargs -I % -n 1 oc delete % -n storage-validation-1 && \
oc get pvc -n storage-validation-1 -o name | xargs -I % -n 1 oc delete % -n storage-validation-1 && \
oc get cm -n storage-validation-1 -o name | xargs -I % -n 1 oc delete % -n storage-validation-1 && \
oc delete ns storage-validation-1 --ignore-not-found && \
oc delete scc zz-fsgroup-scc --ignore-not-found

[INFO ] cleanup script finished with no errors
```

### Verifying your results

Regardless of whether you run the Playbook or use the Container,
on a successful run, you should see the following output:

```
    "msg": "######################## MOUNT TESTS PASSED FOR ReadWriteOnce Volume  #################################"
    "msg": "######################## MOUNT TESTS PASSED FOR ReadWriteMany Volume  #################################"
    "msg": "######################## SEQUENTIAL READ WRITE TEST PASSED FOR ReadWriteOnce Volume #################################"
    "msg": "######################## SEQUENTIAL READ WRITE TEST PASSED FOR ReadWriteMany Volume #################################"
    "msg": "######################## SINGLE THREAD PARALLEL READ WRITE TEST PASSED FOR ReadWriteOnce #################################"
    "msg": "######################## SINGLE THREAD PARALLEL READ WRITE TEST PASSED FOR ReadWriteMany #################################"
    "msg": "######################## MULTI NODE PARALLEL READ WRTIE TEST PASSED FOR ReadWriteOnce #################################"
    "msg": "######################## MULTI NODE PARALLEL READ WRTIE TEST PASSED FOR ReadWriteMany #################################"
    "msg": "######################## FILE UID TEST PASSED FOR ReadWriteMany Volume #################################"
    "msg": "######################## FILE PERMISSIONS TEST PASSED FOR ReadWriteMany Volume #################################"
    "msg": "######################## FILE PERMISSIONS TEST PASSED FOR ReadWriteOnce Volume #################################"
    "msg": "######################## SUB PATH TEST PASSED FOR ReadWriteMany Volume #################################"
    "msg": "######################## FILE LOCK FCNTL TESTS PASSED FOR ReadWriteMany Volume #################################"
    "msg": "######################## FILE LOCK TESTS PASSED FOR ReadWriteMany Volume #################################"
```

```
 PLAY RECAP *********************************************************************
 localhost                  : ok=109  changed=42   unreachable=0    failed=0    skipped=7    rescued=0    ignored=0   
```

## Clean-up Resources

Delete the kuberbetes namespace that you created in [Setup](#setup), you can also run these commands to clean up the
resources in the namespace

```
export STORAGE_VALIDATION_NS=<storage_validation_namespace>

oc delete job $(oc get jobs -n $STORAGE_VALIDATION_NS | awk '{ print $1 }' | grep -i readiness) -n $STORAGE_VALIDATION_NS
oc delete cm $(oc get cm -n $STORAGE_VALIDATION_NS | awk '{ print $1 }' | grep "consumer-\|producer-") -n $STORAGE_VALIDATION_NS
oc delete pvc $(oc get pvc -n $STORAGE_VALIDATION_NS | awk '{ print $1 }' | grep -i readiness) -n $STORAGE_VALIDATION_NS
oc delete scc zz-fsgroup-scc
```

OR

```
export STORAGE_VALIDATION_NS=<storage_validation_namespace>

oc delete project $STORAGE_VALIDATION_NS
oc delete scc zz-fsgroup-scc
```

OR

you can use the provided cleanup script to generate all the commands for the needed cleanup 
(Note that: if you see `No resources found in...`, then no cleanup is needed).

```
USAGE:
  cleanup.sh --namespace <namespace> [--delete-namespace]
``` 
