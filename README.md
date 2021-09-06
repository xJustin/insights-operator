# Insights Operator

This cluster operator gathers anonymized system configuration and reports it to Red Hat Insights. It is a part of the standard OpenShift distribution. The data collected allows for debugging in the event of cluster failures or unanticipated errors.

# Table of Contents

- [Insights Operator](#insights-operator)
- [Table of Contents](#table-of-contents)
- [Building](#building)
- [Testing](#testing)
- [Documentation](#documentation)
- [Getting metrics from Prometheus](#getting-metrics-from-prometheus)
  - [Generate the certificate and key](#generate-the-certificate-and-key)
  - [Prometheus metrics provided by Insights Operator](#prometheus-metrics-provided-by-insights-operator)
    - [Running IO locally](#running-io-locally)
    - [Running IO on K8s](#running-io-on-k8s)
  - [Getting the data directly from Prometheus](#getting-the-data-directly-from-prometheus)
  - [Debugging Prometheus metrics without valid CA](#debugging-prometheus-metrics-without-valid-ca)
- [Debugging](#debugging)
  - [Using the profiler](#using-the-profiler)
    - [Starting IO with the profiler](#starting-io-with-the-profiler)
    - [Collect profiling data](#collect-profiling-data)
    - [Analyzing profiling data](#analyzing-profiling-data)
- [Changelog](#changelog)
  - [Updating the changelog](#updating-the-changelog)
- [Reported data](#reported-data)
  - [Insights Operator Archive](#insights-operator-archive)
    - [Sample IO archive](#sample-io-archive)
    - [Generating a sample archive](#generating-a-sample-archive)
    - [Formatting archive json files](#formatting-archive-json-files)
    - [Obfuscating an archive](#obfuscating-an-archive)
    - [Updating the sample archive](#updating-the-sample-archive)
- [Contributing](#contributing)
- [Support](#support)
- [License](#license)

# Building

To build the operator, install Go 1.11 or above and run:

```shell script
make build
```

To test the operator against a remote cluster, run:

```shell script
bin/insights-operator start --config=config/local.yaml --kubeconfig=$KUBECONFIG
```

where `$KUBECONFIG` has sufficiently high permissions against the target cluster.

# Testing

Unit tests can be started by the following command:

```shell script
make test
```

It is also possible to specify CLI options for Go test. For example, if you need to disable test results caching, use the following command:

```shell script
VERBOSE=-count=1 make test
```

> Integration (e2e) tests are not part of this repository, you can find it [here](https://gitlab.cee.redhat.com/ccx/insights-operator-tests).

# Documentation


The document [docs/gathered-data](docs/gathered-data.md) contains the list of collected data and the API that is used to collect it. This documentation is generated by the command bellow, by collecting the comment tags located above each Gather method.

To start generating the document run:

```shell script
make docs
```

# Getting metrics from Prometheus

## Generate the certificate and key

Certificate and key are required to access Prometheus metrics (instead 404 Forbidden is returned). It is possible to generate these two files from Kubernetes config file. Certificate is stored in `users/admin/client-cerfificate-data` and key in `users/admin/client-key-data`. Please note that these values are encoded by using Base64 encoding, so it is needed to decode them, for example by `base64 -d`.

There's a tool named `gen_cert_key.py` that can be used to automatically generate both files. It is stored in `tools` subdirectory.

```shell script
gen_cert_file.py kubeconfig.yaml
```

## Prometheus metrics provided by Insights Operator

It is possible to read Prometheus metrics provided by Insights Operator. Example of metrics exposed by Insights Operator can be found at [metrics.txt](docs/metrics.txt)

Depending on how or where the IO is running you may have different ways to retrieve the metrics. Here is a list of some options, so you can find the one that fits you:

### Running IO locally

If the IO runs locally, the following command migth be used:

```shell script
curl --cert k8s.crt --key k8s.key -k https://localhost:8443/metrics
```

### Running IO on K8s

Get the token

```shell script
oc whoami -t
```

Read metrics from Pod

```shell script
oc exec \
    -it deployment/insights-operator \
    -n openshift-insights -- \
    curl -k -H "Authorization: Bearer YOUR-TOKEN-HERE" 'https://localhost:8443/metrics'
```

## Getting the data directly from Prometheus

```shell script
sudo kubefwd svc -n openshift-monitoring -d openshift-monitoring.svc -l prometheus=k8s
curl --cert k8s.crt --key k8s.key  -k 'https://prometheus-k8s.openshift-monitoring.svc:9091/metrics'
```

## Debugging Prometheus metrics without valid CA

Get the token

```shell script
oc sa get-token prometheus-k8s -n openshift-monitoring
```

Change in `pkg/controller/operator.go` after creating `metricsGatherKubeConfig` (about line #86)

```go
metricsGatherKubeConfig.Insecure = true
metricsGatherKubeConfig.BearerToken = "YOUR-TOKEN-HERE"
# by default CAFile is /var/run/secrets/kubernetes.io/serviceaccount/service-ca.crt
metricsGatherKubeConfig.CAFile = ""
metricsGatherKubeConfig.CAData = []byte{}
```

# Debugging

## Using the profiler

### Starting IO with the profiler

IO starts a profiler if given the correct environment.
Set the `OPENSHIFT_PROFILE` env variable to "web".

```shell script
export OPENSHIFT_PROFILE=web
```

### Collect profiling data

After IO starts the profiling can be accessed at `http://localhost:6060`, you can use the `pprof` tool to connect to it.

Some profiling examples:

```shell script
# CPU profiling for 30 seconds
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
```

```shell script
# heap profiling
go tool pprof http://localhost:6060/debug/pprof/heap
```

These commands will create a compressed file that can be visualized using a variety of tools, one of them is the `pprof` tool.

### Analyzing profiling data

Starting a web ui at `localhost:8080` to visualize/analyze the profiling data:

```shell script
go tool pprof -http=:8080 /path/to/profiling.out
```

For extra info: [check this link](https://jvns.ca/blog/2017/09/24/profiling-go-with-pprof/)

# Changelog

You can find the project changelog by [clicking here](CHANGELOG.md).

## Updating the changelog

At `./cmd/changelog/main.go` there is a script that can update the changelog for you.

It uses both the local git and GitHub`s API to update the file so:

- To get info from GitHub you will need to set the `GITHUB_TOKEN` envvar to a GitHub access-token.
- Make sure that you have a local, up-to-date copy of each release-branch that might be in the changelog.

It can be used 2 ways:

1. Providing no command line arguments the script will update the current `CHANGELOG.md` with the latest changes according to the local git state.

> 🚨 IMPORTANT: It will only work with changelogs created with this script

```shell script
go run cmd/changelog/main.go
```

2. Providing 2 command line arguments, `AFTER` and `UNTIL` dates the script will generate a new `CHANGELOG.md` within the provided time frame.

```shell script
go run cmd/changelog/main.go 2021-01-10 2021-01-20
```

# Reported data

* ClusterVersion
* ClusterOperator objects
* All non-secret global config (hostnames and URLs anonymized)

The list of all collected data with description, location in produced archive and link to Api and some examples is at [docs/gathered-data.md](docs/gathered-data.md)

The resulting data is packed in `.tar.gz` archive with folder structure indicated in the document. Example of such archive is at [docs/insights-archive-sample](docs/insights-archive-sample).

## Insights Operator Archive

### Sample IO archive

There is a sample IO archive maintained in this repo to use as a quick reference. (can be found at [docs/insights-archive-sample](https://github.com/openshift/insights-operator/tree/master/docs/insights-archive-sample))

To keep it up-to-date it is **required** to update this manually when developing a new data enhancement.

Make sure the `.json` files are in a humanly readable format in the sample archive.
By doing this its easier to review a data enhancement PR, and rule developers can easily check what data it collects.

### Generating a sample archive

Run the insights-operator on a test cluster (from `cluster-bot` or `Quicklab` or etc).

### Formatting archive json files

This formats `.json` files from folder with extracted archive.

```shell script
find . -type f -name '*.json' -print | while read line; do cat "$line" | jq > "$line.tmp" && mv "$line.tmp" "$line"; done
```

### Obfuscating an archive

You can run obfuscation with an archive by running the next command:

```shell script
go run ./cmd/obfuscate-archive/main.go YOUR_ARCHIVE.tar.gz
```

where `YOUR_ARCHIVE.tar.gz` is the path to the archive.
The obfuscated version will be created in the same directory and called `YOUR_ARCHIVE-obfuscated.tar.gz`

### Updating the sample archive

The `docs/insights-archive-sample/` directory contains an example of an Insights
Operator archive, extracted and with pretty-formatted JSON files.
In case of any changes that affect multiple files in the archive, it is a good
idea to regenerate the sample archive to make sure it remains up-to-date.

There are two ways of updating the sample archive directory automatically.
Both of them require running the Insights Operator, letting it generate an archive
and extracting the archive into an otherwise empty directory.

The script will automatically replace existing files in the sample archive with
their respective counterparts from the supplied extracted IO archive.
In case of files with (partially) randomized names, such as pods or nodes,
the entire directory is deleted and replaced with a matching directory from
the new archive if possible.
Changes made by the script can be checked and reverted using Git.
The updated JSON files will be automatically pretty-formatted using `jq`,
which is the only dependency required for running the script.

All existing files in the sample archive can be updated using the following command:

```sh
./scripts/update_sample_archive.sh <Path of directory with the NEW extracted IO archive>
```

If you only want to update files containing a certain string pattern,
you can supply a regular expression as a second optional argument.
For example, the following command was used to replace JSON files containing
the `managedFields` field when it was removed from the IO archive to save space:

```sh
./scripts/update_sample_archive.sh <Path of directory with the NEW extracted IO archive> '"managedFields":'
```

The path of the sample archive directory should be constant relative to
the path of the script and therefore does not have to be specified explicitly.

# Contributing

See [CONTRIBUTING](CONTRIBUTING.md) for workflow & convention details.

See [STYLEGUIDE](STYLEGUIDE.md) for file format and coding style guide.

# Support

Insights Operator is part of Red Hat OpenShift Container Platform. For product-related issues, please
file a ticket [in Red Hat Bugzilla](https://bugzilla.redhat.com/enter_bug.cgi?product=OpenShift%20Container%20Platform&component=Insights%20Operator) for "Insights Operator" component.

# License

This project is licensed by the Apache License 2.0. For more information check the LICENSE file.
