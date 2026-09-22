# OCI Always Free A1 Instance Monitor — GitHub Actions

A GitHub Actions automation project that repeatedly asks **Oracle Cloud Infrastructure (OCI)** for an **Always Free Ampere A1 VM** when the requested capacity is temporarily unavailable.

## What this project is for

This project was built for a personal cloud-server use case. The intended end goal is to obtain an Ubuntu ARM64/AArch64 OCI VM that can later be used as a **personal V2Ray server** for a small number of authorized users/devices.

**Important:** this repository only provisions the OCI virtual machine. It does **not** install, configure, or manage V2Ray. After the VM is created, you are responsible for configuring software on that server and following your provider's terms and applicable rules.

The monitor is designed around this target:

```text
Region:               ap-singapore-2 (Singapore West)
Availability Domain:  YQld:AP-SINGAPORE-2-AD-1
Shape:                VM.Standard.A1.Flex
OCPU:                 1
Memory:               6 GB
Boot volume:          50 GB
Public IPv4:          Enabled through the selected subnet
Image:                Newest standard Ubuntu ARM64 / aarch64 image discovered from OCI
```

Oracle's current Always Free documentation lists **1,500 OCPU-hours + 9,000 GB-hours/month** for A1, equivalent to **2 OCPUs + 12 GB RAM** when used continuously, and up to **10 TB/month outbound data transfer**. Always Free compute must be created in the tenancy's home region. [Oracle Always Free Resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)

---

# 1. How the automation works

There is intentionally **no 6-hour cron schedule** in this version.

The workflow is manually started once, then each 3-hour polling window requests exactly one successor workflow when the current window finishes normally.

```text
Start workflow manually
        |
        v
3-hour polling window
        |
        +--> Check whether target VM already exists
        |
        +--> If no VM, request 1 OCPU / 6 GB A1
        |
        +--> OutOfHostCapacity?
        |       |
        |       +--> wait 120 seconds
        |       +--> retry
        |
        +--> Success / VM detected?
        |       |
        |       +--> STOP PERMANENTLY
        |
        +--> 3 hours reached with no VM
                |
                +--> request ONE successor workflow
                        |
                        v
                    next 3-hour window
```

The workflow uses a concurrency group so only one provisioning run executes at a time. The next run may briefly remain queued while the current run exits.

**Important:** GitHub controls runner allocation, so an exact 10-second wall-clock handoff cannot be guaranteed. The 10-second handoff is an intentional minimum buffer, not a guaranteed scheduling SLA.

GitHub's current documentation states that standard GitHub-hosted runners are free for public repositories. GitHub also provides workflow `permissions` controls so a workflow receives only the GitHub token permissions it actually needs. See:

- https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax
- https://docs.github.com/en/actions/concepts/billing-and-usage

---

# 2. Repository structure

The repository should look like this:

```text
oracle-free-instance-monitor/
|
+-- .github/
|   \-- workflows/
|       \-- OCI_initialize.yml
|
+-- Server/
|   \-- Script.py
|
+-- .gitignore
+-- LICENSE
+-- README.md
\-- requirements.txt
```

Do **not** put API private keys or SSH private keys in this repository.

---

# 3. What you need before starting

You need:

1. An OCI tenancy.
2. An OCI user/group for automation.
3. An OCI API signing key for that automation user.
4. An OCI VCN and subnet.
5. A subnet that permits public IP assignment.
6. An SSH public key.
7. A GitHub account.
8. A public GitHub repository.

The example configuration in this repository is for **Singapore West**:

```text
OCI region:            ap-singapore-2
Availability Domain:   YQld:AP-SINGAPORE-2-AD-1
```

Your own tenancy may use a different region/AD; see the configuration-change section later in this README.

---

# 4. OCI setup — beginner guide

## 4.1 Create a dedicated automation group

In OCI Console:

```text
Navigation Menu
-> Identity & Security
-> Domains
-> your identity domain
-> Groups
```

Create:

```text
Group name: GitHubOCIProvisioners
```

Description example:

```text
Limited group used by GitHub Actions for OCI instance provisioning.
```

---

## 4.2 Create the automation user

Go to:

```text
Identity & Security
-> Users
-> Create User
```

Example:

```text
First name: GitHub
Last name: OCI Provisioner
Username: github-oci-provisioner
```

This user is intended for API automation rather than normal human console use.

---

## 4.3 Add the user to the group

Open:

```text
Identity & Security
-> Users
-> github-oci-provisioner
-> Groups
-> Add User to Group
```

Select:

```text
GitHubOCIProvisioners
```

The relationship should be:

```text
GitHubOCIProvisioners
        |
        +-- github-oci-provisioner
```

---

# 5. OCI IAM policy

Go to:

```text
Identity & Security
-> Policies
-> Create Policy
```

Example name:

```text
GitHubOCIProvisionerPolicy
```

For the simple tenancy-wide setup used by this repository, the policy statements are:

```text
Allow group GitHubOCIProvisioners to manage instance-family in tenancy
Allow group GitHubOCIProvisioners to use volume-family in tenancy
Allow group GitHubOCIProvisioners to use virtual-network-family in tenancy
Allow group GitHubOCIProvisioners to read app-catalog-listing in tenancy
```

### Why these permissions are used

```text
instance-family       -> create/manage Compute instances
volume-family         -> use boot/block volumes during the launch
virtual-network-family-> use the VCN/subnet/VNIC needed by the VM
app-catalog-listing   -> read image/catalog listings used by provisioning
```

For a production setup, you can reduce the scope to specific compartments instead of `in tenancy`. Oracle's policy documentation describes compartment-scoped Compute launch policies.

Oracle policy documentation:
https://docs.oracle.com/en-us/iaas/Content/Identity/Concepts/commonpolicies.htm

---

# 6. Create the OCI API signing key

Open:

```text
Identity & Security
-> Users
-> github-oci-provisioner
-> API Keys
-> Add API Key
```

Generate an API key pair.

Download the private key and store it safely on your computer.

**Never upload the `.pem` private key to this public repository.**

You will need these values later:

```text
Automation User OCID
API key fingerprint
Tenancy OCID
Private key PEM contents
```

OCI API signing-key documentation:
https://docs.oracle.com/en-us/iaas/Content/API/Concepts/apisigningkey.htm

---

# 7. Find the OCI subnet OCID

Go to:

```text
Networking
-> Virtual Cloud Networks
-> your VCN
-> Subnets
-> your subnet
```

Copy the **Subnet OCID**.

It looks similar to:

```text
ocid1.subnet.oc1....
```

Do not post the real value publicly.

Also confirm the subnet allows public IP assignment. In the subnet details, the setting must not prohibit public IPs.

This repository checks the subnet before starting the polling loop.

---

# 8. Prepare an SSH public key

The SSH key used for the VM is separate from the OCI API signing key.

On Windows PowerShell, check whether you already have an SSH key:

```powershell
Get-ChildItem $env:USERPROFILE\.ssh
```

If you have:

```text
id_ed25519
id_ed25519.pub
```

display the public key:

```powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub
```

Copy the complete `ssh-ed25519 ...` line.

Use the `.pub` file only. Never expose the private key.

---

# 9. Create the GitHub repository

Create a new repository, for example:

```text
oracle-free-instance-monitor
```

Set:

```text
Visibility: Public
```

The project is intentionally public so standard GitHub-hosted Actions runners can be used without the monthly GitHub Free private-repository Actions-minute quota. GitHub documents standard GitHub-hosted runner usage as free for public repositories.

**Do not treat the public repository as a place to store secrets.**

---

# 10. GitHub Actions Secrets — all 7 values

Open:

```text
Repository
-> Settings
-> Secrets and variables
-> Actions
-> New repository secret
```

Create these seven repository secrets:

```text
OCI_USER_ID
OCI_PRIVATE_KEY
OCI_FINGERPRINT
OCI_TENANCY_ID
OCI_REGION
OCI_SUBNET_ID
OCI_PUBLIC_SSH_KEY
```

## 10.1 OCI_USER_ID

Value:

```text
The OCID of github-oci-provisioner
```

Do not use the human administrator username here.

---

## 10.2 OCI_PRIVATE_KEY

Open the `.pem` file downloaded from OCI.

Paste the **entire** private key, including:

```text
-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----
```

Do not add the secret name to the value.

---

## 10.3 OCI_FINGERPRINT

Value:

```text
The fingerprint of the automation user's OCI API signing key
```

---

## 10.4 OCI_TENANCY_ID

Value:

```text
Your OCI tenancy OCID
```

---

## 10.5 OCI_REGION

For this example repository:

```text
ap-singapore-2
```

---

## 10.6 OCI_SUBNET_ID

Value:

```text
The OCID of the subnet that should receive the VM's VNIC
```

---

## 10.7 OCI_PUBLIC_SSH_KEY

Paste the complete single-line SSH **public** key, for example:

```text
ssh-ed25519 AAAAC3... comment
```

Do not paste `id_ed25519` or another SSH private-key file.

---

# 11. GitHub workflow permissions

This repository uses:

```yaml
permissions:
  contents: read
  actions: write
```

`contents: read` lets the workflow check out the repository.

`actions: write` is required because the current design requests exactly one successor `workflow_dispatch` run after a normal 3-hour polling window.

GitHub documents `GITHUB_TOKEN` permissions here:
https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax

Do **not** add broad permissions such as:

```yaml
permissions: write-all
```

---

# 12. Current server target — exact configuration

The current repository is intentionally simple. It targets **only one VM configuration**:

```text
Region:               ap-singapore-2 (Singapore West)
Availability Domain:  YQld:AP-SINGAPORE-2-AD-1
Shape:                VM.Standard.A1.Flex
OCPU:                 1
Memory:               6 GB
Boot volume:          50 GB
Public IPv4:          enabled
Image:                newest standard Ubuntu ARM64/aarch64 image discovered from OCI
```

For your intended small personal V2Ray use, the VM is simply the server foundation. The V2Ray software itself is outside this repository.

---

# 13. IMPORTANT — where to change each target setting

This section is intended as a quick reference for anyone modifying the project.

## 13.1 Change the region

### File

```text
Server/Script.py
```

### Find

```python
EXPECTED_REGION = "ap-singapore-2"
```

### Change to

```python
EXPECTED_REGION = "YOUR-OCI-REGION"
```

For example:

```python
EXPECTED_REGION = "ap-mumbai-1"
```

**Also change the GitHub Secret**:

```text
OCI_REGION
```

so that it contains the same region identifier.

### Important

The script intentionally verifies that the GitHub secret and `EXPECTED_REGION` match:

```python
if config["region"] != EXPECTED_REGION:
    print("❌ REGION MISMATCH")
    ...
```

If you change only one of them, the workflow stops.

Also make sure the new region is actually available to the tenancy and is allowed for the account's intended resources.

---

## 13.2 Change the Availability Domain

### File

```text
Server/Script.py
```

### Find

```python
EXPECTED_AD = "YQld:AP-SINGAPORE-2-AD-1"
```

### Change to

```python
EXPECTED_AD = "YOUR-AVAILABILITY-DOMAIN"
```

The script first calls OCI to discover the tenancy's ADs and then checks that the configured `EXPECTED_AD` exists.

For this Singapore-West tenancy, the expected AD is:

```text
YQld:AP-SINGAPORE-2-AD-1
```

Do not invent an AD name. Copy the exact value returned by OCI.

---

## 13.3 Change the Compute shape

### File

```text
Server/Script.py
```

### Find

```python
SHAPE = "VM.Standard.A1.Flex"
```

### Example

```python
SHAPE = "VM.Standard.A1.Flex"
```

If you use a different shape, verify that the chosen Ubuntu image architecture and your intended Always Free/paid status are compatible.

For this project, keep A1 unless you intentionally want to redesign the image and capacity logic.

---

## 13.4 Change OCPU count

### File

```text
Server/Script.py
```

### Find

```python
OCPUS = 1
```

### Example for 2 OCPUs

```python
OCPUS = 2
```

The OCPU value is used here:

```python
shape_config = oci.core.models.LaunchInstanceShapeConfigDetails(
    ocpus=OCPUS,
    memory_in_gbs=MEMORY_GB,
)
```

For Always Free A1, Oracle currently documents a total of **2 OCPUs and 12 GB RAM** per tenancy allocation. Using more than the Always Free allocation may create paid usage on an eligible paid tenancy.

Oracle Always Free resources:
https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm

---

## 13.5 Change memory

### File

```text
Server/Script.py
```

### Find

```python
MEMORY_GB = 6
```

### Example for the full current A1 Always Free allocation

```python
MEMORY_GB = 12
```

The memory is passed here:

```python
shape_config = oci.core.models.LaunchInstanceShapeConfigDetails(
    ocpus=OCPUS,
    memory_in_gbs=MEMORY_GB,
)
```

Keep the OCPU and memory combination valid for the selected OCI shape and account limits.

---

## 13.6 Change the boot volume size

### File

```text
Server/Script.py
```

### Find

```python
BOOT_VOLUME_GB = 50
```

### Example for 100 GB

```python
BOOT_VOLUME_GB = 100
```

The value is applied here:

```python
source_details = oci.core.models.InstanceSourceViaImageDetails(
    source_type="image",
    image_id=image_id,
    boot_volume_size_in_gbs=BOOT_VOLUME_GB,
)
```

Make sure the requested storage remains within the Always Free allocation if your objective is a $0 Always Free instance.

---

## 13.7 Enable/disable a public IPv4 address

### File

```text
Server/Script.py
```

### Find

```python
create_vnic_details=oci.core.models.CreateVnicDetails(
    subnet_id=subnet_id,
    assign_public_ip=True,
    assign_private_dns_record=True,
    display_name=VNIC_NAME,
),
```

### Public IPv4 enabled

```python
assign_public_ip=True
```

### Public IPv4 disabled

```python
assign_public_ip=False
```

For the current server purpose, public IPv4 is enabled because the VM is intended to be remotely reachable.

The selected subnet must also allow public IP assignment. The script checks:

```python
if subnet.prohibit_public_ip_on_vnic:
    raise RuntimeError(...)
```

If the subnet prohibits public IPs, changing only `assign_public_ip=True` is not enough; you need a subnet that permits public addressing.

---

## 13.8 Change the Ubuntu image selection

This project intentionally does **not** hard-code an Ubuntu image OCID.

### File

```text
Server/Script.py
```

### Function

```python
find_a1_ubuntu_image(...)
```

The selection logic looks for:

```python
if "minimal" in name:
    continue

if "aarch64" not in name and "arm64" not in name:
    continue
```

and then prefers Ubuntu 24.04:

```python
if "24.04" in name:
    version_score = 2
elif "22.04" in name:
    version_score = 1
```

### Current behavior

```text
OCI image catalog
      |
      v
standard Ubuntu images
      |
      v
ARM64 / aarch64 only
      |
      v
prefer Ubuntu 24.04
      |
      v
newest matching image
```

This is safer than hard-coding an old image OCID because OCI platform images are refreshed over time.

### If you intentionally want to force a specific image

You could replace the automatic selection with a fixed OCID, but this is **not recommended** for a long-running monitor because the image may become unavailable later.

The hard-coded-image alternative would conceptually be:

```python
IMAGE_ID = "ocid1.image.oc1...."
```

and then use `IMAGE_ID` when building the launch request.

Do not put credentials in an image ID or replace the GitHub Secrets with hard-coded secrets.

---

# 14. Change the retry interval

### File

```text
Server/Script.py
```

Find:

```python
RETRY_INTERVAL_SECONDS = 120
```

For example, 180 seconds:

```python
RETRY_INTERVAL_SECONDS = 180
```

The current 120-second value means:

```text
Attempt
  |
  +--> OutOfHostCapacity
  |
  +--> wait 120 seconds
  |
  +--> next attempt
```

This is a project setting, not an official Oracle guarantee that exactly 120 seconds is required.

The script also handles HTTP 429 separately and waits longer when OCI indicates rate limiting.

---

# 15. Change the polling-window duration

### File

```text
Server/Script.py
```

Find:

```python
MAX_RUN_SECONDS = 180 * 60
```

The current value is:

```text
180 minutes = 3 hours
```

### Example: 2 hours

```python
MAX_RUN_SECONDS = 120 * 60
```

### Example: 4 hours

```python
MAX_RUN_SECONDS = 240 * 60
```

Also review the workflow timeout in:

```text
.github/workflows/OCI_initialize.yml
```

Current value:

```yaml
timeout-minutes: 195
```

Leave enough time for checkout, dependency installation, handoff, and runner overhead. Do not set the Python loop so close to GitHub's hosted-job maximum that a clean handoff becomes unreliable.

---

# 16. Change the instance name

### File

```text
Server/Script.py
```

Find:

```python
INSTANCE_NAME = "FX-Backend-Server"
```

Example:

```python
INSTANCE_NAME = "My-A1-Server"
```

The instance name is also used by the duplicate-protection lookup. If you change it, both creation and detection automatically use the new name because both reference `INSTANCE_NAME`.

---

# 17. Why the script checks for an existing VM before every launch

This is an important safety feature.

A launch request could have an ambiguous response if the network connection fails after OCI receives the request.

The script therefore does:

```text
Check existing instance
        |
        +--> exists -> STOP
        |
        +--> does not exist -> launch request
```

This is especially important after connection timeouts.

Do not remove the existing-instance safety check unless you redesign the provisioning logic carefully.

---

# 18. Error handling

The monitor does not treat every OCI failure as an ordinary capacity retry.

## Retried automatically

```text
OutOfHostCapacity
HTTP 429 rate limiting (with longer backoff)
classified transient server/network conditions handled by the script
```

## Stop and investigate

```text
401 Authentication
403 Authorization
400 Invalid request
404 Missing resource
invalid subnet
invalid image
invalid configuration
```

This prevents an IAM mistake from turning into an endless API loop.

---

# 19. GitHub Actions workflow

File:

```text
.github/workflows/OCI_initialize.yml
```

The workflow intentionally has:

```yaml
on:
  workflow_dispatch:
```

There is no six-hour cron schedule in this design.

The workflow uses:

```yaml
concurrency:
  group: oci-a1-provisioner
  cancel-in-progress: false
```

This prevents the successor run from cancelling the current run.

The workflow also uses:

```yaml
permissions:
  contents: read
  actions: write
```

`actions: write` is required for the self-dispatch handoff.

Do not change this to broad `write-all` permissions.

---

# 20. How to start the monitor

After the files and all seven GitHub Secrets are configured:

1. Open the repository.
2. Click **Actions**.
3. Select **OCI Always Free A1 Instance Monitor**.
4. Click **Run workflow**.
5. Confirm the branch is `main`.
6. Start the workflow.

The first run should show approximately:

```text
OCI ALWAYS FREE A1 INSTANCE PROVISIONER
Region              : ap-singapore-2
Availability Domain : YQld:AP-SINGAPORE-2-AD-1
Shape               : VM.Standard.A1.Flex
OCPUs               : 1
Memory              : 6 GB
```

Then you should see:

```text
AUTHENTICATION CHECK
✅ OCI authentication succeeded.

AVAILABILITY DOMAIN CHECK

SUBNET CHECK
✅ Subnet can assign a public IPv4 address.

INITIAL INSTANCE CHECK
✅ No existing target instance found.

IMAGE DISCOVERY
✅ Selected Ubuntu ARM image:
    Name : Canonical-Ubuntu-24.04-aarch64-....

STARTING CAPACITY POLLING

[Attempt 1] VM.Standard.A1.Flex / 1 OCPU / 6 GB
```

If OCI responds with:

```text
Out of host capacity
```

the script waits 120 seconds and tries again.

---

# 21. What success looks like

When OCI finally accepts the request:

```text
🎉 INSTANCE CREATION REQUEST ACCEPTED

Instance ID     : ocid1.instance....
Display name    : FX-Backend-Server
Lifecycle state : PROVISIONING
Shape           : VM.Standard.A1.Flex
OCPUs           : 1
Memory          : 6 GB

Oracle accepted the launch request.
🛑 Provisioning loop stopped immediately.
```

The Python program reports:

```text
status=DONE
```

The workflow therefore does not dispatch another polling window.

---

# 22. What happens if the 3-hour window ends

If no VM is created during the 3-hour window:

```text
RUN WINDOW FINISHED
No instance was created during this workflow run.
This polling window finished normally.
The workflow will request exactly one fresh run.
```

Then the workflow:

```text
waits 10 seconds
     |
     v
requests one fresh workflow_dispatch run
     |
     v
current run exits
```

The next run starts when GitHub makes the next runner available.

The exact 10-second wall-clock handoff is not guaranteed by GitHub.

---

# 23. What happens after the VM already exists

Suppose a previous run successfully created the VM.

On any later run, the script finds:

```text
FX-Backend-Server
```

and immediately reports:

```text
✅ Target instance already exists.
🛑 Provisioning stopped.
```

This prevents the monitor from continuously creating additional instances.

---

# 24. Troubleshooting

## Error: `401` or `403`

Likely causes:

- wrong OCI user OCID
- wrong fingerprint
- invalid private key
- user not in `GitHubOCIProvisioners`
- insufficient IAM policy

Do not keep retrying. Fix authentication/permissions first.

---

## Error: `Out of host capacity`

Example:

```text
HTTP status : 500
OCI code    : InternalError
Message     : Out of host capacity.
```

This means OCI did not have the requested Always Free host capacity at that moment.

The script intentionally waits 120 seconds and retries.

Oracle also documents waiting or trying another availability domain when available, and notes that PAYG/paid accounts provide access to additional paid resources. [Oracle Always Free Resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)

For this example tenancy, only the documented Singapore West AD is targeted.

---

## Error: no Ubuntu aarch64 image found

The script should normally discover a current standard Ubuntu ARM image automatically.

If it fails:

1. Open OCI Compute image listings manually.
2. Confirm a standard Ubuntu ARM64/aarch64 platform image is available in the region.
3. Check that the script's local filtering has not been made too restrictive.

Do not immediately hard-code an old image OCID without understanding why image discovery failed.

---

## Error: `ConnectTimeoutError`

This means the GitHub runner could not connect to the OCI API endpoint within the configured connection timeout.

A temporary connection problem is different from `OutOfHostCapacity`.

The safe retry strategy is:

```text
connection/network error
        |
        v
wait/backoff
        |
        v
check whether the VM exists
        |
        v
retry only if it does not exist
```

Do not remove the duplicate-protection check after a timeout.

---

## Error: HTTP `429`

This indicates API throttling/rate limiting.

The script intentionally waits longer than the normal 120-second capacity interval for a 429 rather than increasing the request rate.

---

# 25. Public repository security checklist

Because this repository is public:

- Never commit `OCI_PRIVATE_KEY`.
- Never commit an SSH private key.
- Never hard-code API credentials.
- Keep OCI secrets only in GitHub Actions Secrets.
- Protect the `main` branch.
- Do not merge untrusted modifications to `Server/Script.py` or `.github/workflows/OCI_initialize.yml`.
- Do not add `pull_request` or other untrusted-code triggers to the credential-bearing workflow unless you understand the security consequences.
- Keep `permissions:` as narrow as practical.
- Do not print environment variables containing secrets.

GitHub security documentation:
https://docs.github.com/en/actions/reference/security/secure-use

GitHub Secrets documentation:
https://docs.github.com/en/actions/concepts/security/secrets

---

# 26. How to migrate this repository to another OCI tenancy

If someone wants to reuse this project for their own OCI account, the code can remain mostly unchanged.

They should update:

```text
GitHub Secrets
    OCI_USER_ID
    OCI_PRIVATE_KEY
    OCI_FINGERPRINT
    OCI_TENANCY_ID
    OCI_REGION
    OCI_SUBNET_ID
    OCI_PUBLIC_SSH_KEY
```

and, if their region/AD differs, update:

```python
EXPECTED_REGION = "..."
EXPECTED_AD = "..."
```

in:

```text
Server/Script.py
```

Then verify the target shape/image combination.

---

# 27. Quick configuration reference

| Target | Current value | Where to change |
|---|---|---|
| Region | `ap-singapore-2` | `Server/Script.py` -> `EXPECTED_REGION` + GitHub Secret `OCI_REGION` |
| Availability Domain | `YQld:AP-SINGAPORE-2-AD-1` | `Server/Script.py` -> `EXPECTED_AD` |
| Shape | `VM.Standard.A1.Flex` | `Server/Script.py` -> `SHAPE` |
| OCPU | `1` | `Server/Script.py` -> `OCPUS` |
| Memory | `6` GB | `Server/Script.py` -> `MEMORY_GB` |
| Boot volume | `50` GB | `Server/Script.py` -> `BOOT_VOLUME_GB` |
| Public IPv4 | enabled | `assign_public_ip=True` in `build_launch_details()` |
| V2Ray purpose | personal server target | documentation only; V2Ray is not installed by this repo |
| Ubuntu image | newest standard ARM64 | `find_a1_ubuntu_image()` discovery logic |
| Retry interval | `120` seconds | `RETRY_INTERVAL_SECONDS` |
| Polling window | `180` minutes | `MAX_RUN_SECONDS` |
| Instance name | `FX-Backend-Server` | `INSTANCE_NAME` |

---

# 28. Reference documentation

## Oracle Cloud

- Always Free resources: https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm
- API signing keys: https://docs.oracle.com/en-us/iaas/Content/API/Concepts/apisigningkey.htm
- IAM common policies: https://docs.oracle.com/en-us/iaas/Content/Identity/Concepts/commonpolicies.htm
- Compute shapes: https://docs.oracle.com/en-us/iaas/Content/Compute/References/computeshapes.htm
- Ubuntu images: https://docs.oracle.com/en-us/iaas/Content/Compute/References/images.htm

## GitHub

- Workflow syntax: https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax
- Billing and usage: https://docs.github.com/en/actions/concepts/billing-and-usage
- Secrets: https://docs.github.com/en/actions/concepts/security/secrets
- Secure use: https://docs.github.com/en/actions/reference/security/secure-use

---

# 29. Summary

This repository is a **small OCI provisioning monitor** built to solve one specific problem:

```text
OCI Always Free A1 capacity temporarily unavailable
                |
                v
GitHub Actions repeatedly requests the VM
                |
                v
1 OCPU + 6 GB A1
                |
                v
Newest standard Ubuntu ARM64 image
                |
                v
VM created
                |
                v
STOP all further provisioning
```

The intended end use of the VM in this project is a **personal server foundation that can later host V2Ray**, but V2Ray installation/configuration is deliberately outside this repository.

For the default target, do not change the configuration unless you understand OCI's region, shape, image, storage, and Always Free rules.
