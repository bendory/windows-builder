# windows-builder

An experimental Windows builder for Cloud Build.

## Getting started

If you are new to Google Cloud Build, we recommend you start by visiting the [manage resources page](https://console.cloud.google.com/cloud-resource-manager) in the Cloud Console, [learn how to enable billing](https://cloud.google.com/billing/docs/how-to/modify-project), [enable the Cloud Build API](https://console.cloud.google.com/flows/enableapi?apiid=cloudbuild.googleapis.com), and [install the Cloud SDK](https://cloud.google.com/sdk/docs/).

Grant Compute Engine permissions to your Cloud Build service account:

```sh
gcloud services enable compute.googleapis.com
export PROJECT=$(gcloud info --format='value(config.project)')
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT --format 'value(projectNumber)')
export CB_SA_EMAIL=$PROJECT_NUMBER@cloudbuild.gserviceaccount.com
gcloud projects add-iam-policy-binding $PROJECT --member=serviceAccount:$CB_SA_EMAIL --role='roles/compute.admin'  
```

Clone this repository and build the builder:
```sh
gcloud builds submit --config=builder/cloudbuild.yaml builder/
```

Then, refer to the builder in your project's `cloudbuild.yaml`.  To spin up an ephemeral `n1-standard-1` VM on Compute Engine, simply provide the command you wish to execute.  Your Cloud Build workspace is synchronized to `C:\workspace` on the server first.

```yaml
steps:
- name: 'gcr.io/$PROJECT_ID/windows-builder'
  args: [ '--command', '<command goes here>' ]
```

The VM is configured by the builder and then deleted automatically at the end of the build.

To use an existing Windows server instead, also provide the hostname, username and password:

```yaml
steps:
- name: 'gcr.io/$PROJECT_ID/windows-builder'
  args: [ '--hostname', 'host.domain.net',
          '--username', 'myuser',
          '--password', 's3cret',
          '--command', '<command goes here>' ]
```

Your server must support Basic Authentication (username and password) and your network must allow access from the internet on TCP port 5986.

## Performance

Starting an ephemeral VM on Compute Engine takes about 3 min 31 sec, broken down as follows:

| Step | Duration | Elapsed | Fraction | 
|------|----------|---------|----------|
| Create new Compute Engine instance | 7 sec | 7 sec | 3% |
| Wait for password reset | 37 sec | 45 sec | 21% |
| Wait for Windows to finish booting | 2 min 39 sec | 3 min 24 sec | 76% |

Frequent builds will benefit from creating a persistent Windows VM.  To do this, see `scripts/start_windows.sh`.

## Security

`windows-builder` communicates with the remote server using WinRM over HTTPS.  Basic authentication using username and password is currently supported.

For ephemeral VMs on Compute Engine, the initial password reset is performed [using public key cryptography](https://cloud.google.com/compute/docs/instances/windows/automate-pw-generation).  The cleartext password is never sent over an unencrypted connection, and is stored in memory for the duration of the build.  The latest version of Windows Server 1803 DC Core for Containers (released 2018-08-02) is currently used.

## Docker builds

Windows supports [two different types](https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/hyperv-container) of containers: "Windows Server containers", similar to the traditional Linux container, and "Hyper-V containers", which rely on Virtual Machines.  This code uses Windows Server containers, and as a result both the major and minor version of Windows must match between container and host.

The package manager in Windows Server 1803 provides Docker 17.06.  However to run a Docker executable inside a Docker container as Cloud Build typically does, version 17.09 or higher is required to bind-mount the named pipe: see [this pull request](https://github.com/StefanScherer/insider-docker-machine/pull/1).  An example Docker container is provided in `images/dokcer-windows`.
