
# Recommend API Overview
All clients seeking recommendations are insulated from the underlying channels via a REST API: /recommend .
Cincinnati exposes this endpoint and so does a CVO-orbit entity (for simplicity, the design will say
that the CVO exposes this endpoint directly). The CVO endpoint will be referred to as the "cluster endpoint"
for "/recommend".

If "oc" can talk to the cluster endpoint when fulfilling a recommend request using the 
current kubeconfig context, it will prefer it over the Cincinnati endpoint. A query to the
cluster endpoint will trigger the CVO to call the Cincinnati endpoint to obtain most of the
information that needs to be returned to the user. However, the cluster endpoint will decorate 
the data with additional context from the CVO -- like the relevance & impact of known issues with 
respect to the current cluster (future UpdateInformers can participate behind this cluster-local API).

If "oc" cannot communicate with the CVO endpoint, then it must include a "--from=<version>" in its request to the 
public (or specified) Cincinnati instance. The intent here is to allow someone not actively connected to their 
fleet to explore version information. 

The CVO's channel configuration has no influence on the outcome of 
the /recommend response. The CVO's status with reported available updates would also not be relative to this 
setting. The CVO's status would always report when updates were available for each public policy and step.
"candidate" is not a swimlane intended for regular public consumption. It is visible with 
"--policy pre-release", but its only use would be to make available pre-release builds for partners and 
OpenShift.Next related activity. Otherwise, it serves as a staging area waiting for errata to ship.

# Recommend API Implementation Notes
Purely as a method of implementation (invisible to users), there are three new channels:
- early-insights
- ga
- monthly

As today, a build goes into candidate. An errata eventually ships for it, and the release goes into the 
early-insights channel. 48 hours later, it is automatically promoted into ga.

Graph data is enhanced to include CVE and other fix information to support filtering via "--issue" . The 
endpoints do this filtering, so that we don't need to put too much logic in "oc".

The API should be paginated so that not every edge option needs to be evaluated for every call.

To ease the transition to the new API/interface, the "fast" and "stable" channels will remain in place. 

# Insights Output
Recommendations are output alongside update insights. The MVP will include insights from the CVO, but
UpdateInformer plugins can offer additional insights later. Insights are categorized using:
- RELEVANT: Whether the recommendation can conclude if the issue applies to the cluster or described cluster. 
  - "Yes"
  - "No"
  - "-" for possible/unknown.
- SCOPE
  - Performance
  - Security
  - Availability
  - Integrity - Data integrity / loss.
  - Lifecycle - The insight relates the OpenShift software lifecycle.
  - "-" for other.
- SEVERITY:
  - Critical - The issue prevents normal use of the software or a critical function. It may result in data loss or compromise system integrity.
  - Major - The issue significantly impairs the functionality but doesn't completely prevent its use. There might be a workaround available.
  - Minor - The issue is an inconvenience but doesn't significantly affect the software's core functionality.
  - Info  - The insight is informational only.

# Recommend CLI Arguments

```
$ oc adm update recommend --help
Recommends cluster updates. If authenticated with a cluster, update insights will be refined 
with information specific to that cluster based on its detected configuration and environment.  

Usage:
  oc adm update recommend [flags]

Flags:
  --policy		    "default" (default) recommends the latest generally available version. Frequently updated to deliver 
                  the latest security and bug fixes. Exceeds industry standard of monthly updates.
    
                  "monthly" recommends an industry standard monthly update from among default versions. Updates may be
                  recommended more than once a month if critical CVEs have been fixed.
    
                  "early-insights" recommends all supported versions. Clusters running these releases 
                  inform Red Hat's early insights program where telemetry data helps identify risks before
                  versions are made available in the "default" policy.
    
                  "pre-release" recommends all available versions, including those which are not yet supported for 
                  production. Ideal for test environments or evaluating new features.

  --entitlement	  "eus" - Asserts "Extended Update Support" (EUS) entitlement. EUS versions are otherwise not displayed.
                    
                  "lts" - Asserts "Long Term Support" (LTS) entitlement. LTS versions are otherwise not displayed.
                   
  --to 			      "patch" (default) recommends the next patch version of OpenShift (e.g. 4.16.7 -> 4.16.8). Patch 
                  versions introduce bug and stability fixes but do not introduce new features. 

                  "minor" recommends subsequent minor versions of OpenShift (e.g. 4.16.22 -> 4.17.6). Minor versions 
                  introduce new features but may introduce changes requiring new configuration or migration. 
                  More than one update may be required to reach a new minor, but that path will be recommended.
                  
                  "<version>". For example, "--to 4.18" will recommend an update path to reach the specified minor 
                  version.

  --includes      A Red Hat issue identifier. Recommendation will include whether the release includes a change
                  for the indicated issue(s).
                  
  --output|-o     Output format. 
                  "summary" (defatul) Compact, human readable output. 
                  
                  "details" Print all available insight information.
                  
                  "json" to output in JSON format. Inludes all details.

  --more [token]  If no token is specified, interactively list update options until they are exhausted or Ctrl-C.
                  token is used for JSON output continuation 

                  
Disconnected Options:
Provides update insights based on cluster description (no attempt will be made to evaluated an authenticated
cluster). Either --new or --from-version is required.

  --new           Recommend a version for a new installation of OpenShift.
  --from-version  Recommend an update from the specified version. 
  
  --attributes    Refine update insights based on known attributes of a target cluster when not authenticated
                  with that cluster. Accepts a command delimited list of key=value pairs.
                  Keys:
                    platform=aro|osd|vsphere|aws|gcp|...
                    arch=amd64|arm64|s390x|ppc64le|multi

  --update-server OpenShift Update Service Server endpoint (alternative to api.openshift.com). 

```


# Invocation Examples

## Not Connected
In these scenarios, the user has not authenticated with a cluster or is unauthorized to communicate  
with a local CVO /recommend API.

- Tool extraction URLs should be appropriate to the host architecture running `oc`.

### Not Connected - Error

```
$ oc adm update recommend
Unable to communicate with cluster. Authenticate with a cluster as an administrator or use "--help" for 
to review disonnected options. 
```

### Not Connected - New
"--new" requests information for installing a new cluster. If possible, the output includes information 
about how to download the installer binary. Extracting using "oc adm release extract" can only provide
unsigned binaries. 

```
$ oc adm update recommend --new

Version 4.16.14
Release Date: 03/15/20xx
Reason: 
  - OpenShift 4.16 is the latest generally available version of OpenShift. 
  - 4.16.14 is the latest supported release of 4.16.
Information:
  Extract the installer for this version with "oc adm release extract --tools quay.io/openshift-release-dev/ocp-release:4.16.14-x86_64".
  Signed binaries can be downloaded at ...customer portal... 
```

### Not Connected - New w/ More

```
$ oc adm update recommend --new --more

Version 4.16.14
Release Date: 03/15/20xx
Reason: 
  - OpenShift 4.16 is the latest generally available version of OpenShift. 
  - 4.16.14 is the latest supported release of 4.16.
Information:
  Extract the installer for this version with "oc adm release extract --tools quay.io/openshift-release-dev/ocp-release:4.16.14-x86_64"
  Signed binaries can be downloaded at ...customer portal... 


---Press enter to continue----
Available:

Version: 4.16.13
Release Date: ....
Information:
  Extract ...
  
---Press enter to continue----
Crtl-C  
```

### Not Connected - New w/ From

```
$ oc adm update recommend --new --from-version 4.15

Version 4.15.33
Release Date: 03/15/20xx
Reason: 
  - 4.15.33 is the latest supported release of the same minor
Information:
  Extract the installer for this version with "oc adm release extract --tools quay.io/openshift-release-dev/ocp-release:4.15.33-x86_64  
Insights:
  RELEVANT    SCOPE       SEVERITY    DESCRIPTION
  -           Lifecycle   Info        4.15 is not the latest available version of OpenShift.
```

## Connected 
These examples assume that oc can communicate with the cluster endpoint. 
- The text `<server name>` should be substituted with the result of `oc whoami --show-server`.

### Simple Patch
```
$ oc adm update recommend

Version 4.15.33
Release Date: 03/15/20xx
Reason: 
  - <server name> is running 4.15.
  - 4.15.33 is the latest supported release of the same minor version.
Insights:
  RELEVANT    SCOPE       SEVERITY    DESCRIPTION
  -           Lifecycle   Info        4.15 is not the latest available version of OpenShift.
```

### Simple Minor
```
$ oc adm update recommend --to minor

Version 4.16.3
Release Date: 03/15/20xx
Reason: 
  - <server name> is running 4.15.
  - 4.16.3 is the latest supported release of the next minor.
Insights:
  RELEVANT    SCOPE         SEVERITY    DESCRIPTION
  No          Availability  Major       vSphere cluster storage is slow to...
```

### Update Path
```
$ oc adm update recommend --to minor

** An intermediate update is required to reach the recommended minor version **

Version: 4.15.34
Release Date: 03/15/20xx
Considerations:
  * 4.15.34 has a known issue which may be relevant to the cluster. Review issue details before proceeding.
Reason:
  - <server name> is running 4.15.1.
  - 4.15.1 does not support update directly to 4.16.9.
  - Upgrades from 4.15.34 to 4.16.9 are supported.
Insights:
  RELEVANT    SCOPE         SEVERITY    DESCRIPTION
  -           Lifecycle     Info        This is an intermediate version to reach the destination version.


Destination:

Version: 4.16.9
Release Date: 03/15/20xx
Reason:
  - <server name> is running 4.15.
  - 4.16.9 is the latest supported version of the next minor.
  - Upgrades from 4.15.34 to 4.16.9 are supported.
Insights:
  RELEVANT    SCOPE         SEVERITY    DESCRIPTION
  -           Lifecycle     Info        An intermediate update is required to reach this minor.
  Yes         Availability  Major       Workload scheduling...


Alternative Destinations (use "--to <version>" to see desination insights):
- [4.16.8] through [4.16.5]  
- [4.16.3]
```

- Running with `--more` would move back through 4.15 versions. It is possible that multiple intermediate versions
  would be required given enough "more" queries. For example, if they query back to "4.15.2", it may also be unable
  to update directly to 4.16. So the path might be "4.15.2" -> "4.15.34" -> "4.16.8".
- Alternative destinations are listed because `--more` can't move through two streams simultaneously. For example, 
  4.16.9 shows a major issue relevant to the cluster. The customer may want to avoid it by updating to 4.16.8 instead.
