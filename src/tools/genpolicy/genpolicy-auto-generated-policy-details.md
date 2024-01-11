# High level overview

This document describes some of the details of the Kata Agent Policy documents auto-generated by the `genpolicy` tool. See [`genpolicy`](README.md) for general information about the Kata Agent Policy generation tool.

See [Policy contents](../../../docs/how-to/how-to-use-the-kata-agent-policy.md#policy-contents) for an introduction to typical Kata Agent Policy document contents.

## Package name

The name of the Kata Agent Policy [`Rego`](https://www.openpolicyagent.org/docs/latest/policy-language/) package must be `agent_policy`:

```
package agent_policy
```

## Default values

For an introduction to Policy default values, see [default values](../../../docs/how-to/how-to-use-the-kata-agent-policy.md#default-values).

`genpolicy` copies the default values from [`rules.rego`](rules.rego) into the auto-generated Policy. Therefore, all Policies generated using the same `rules.rego` file are using the same default values.

Some of the [ttRPC API](../../libs/protocols/protos/agent.proto) requests are always allowed by the auto-generated Policy. Those requests have a default value of `true` and there aren't any rules associated with them. Examples:

```
default CreateSandboxRequest := true
default DestroySandboxRequest := true
```

Other requests have a default value of `false` and there is at least one [rule](#rules) associated with these requests that allows such requests depending on the request input parameters. Examples:

```
default CopyFileRequest := false
default CreateContainerRequest := false
```

## Rules

For an introduction to Policy rules, see [rules](../../../docs/how-to/how-to-use-the-kata-agent-policy.md#rules).

`genpolicy` copies the rules from [`rules.rego`](rules.rego) into the auto-generated Policy. Therefore, all Policies generated using the same `rules.rego` file are using the same rules.

For additional details about the `genpolicy` rules see [`genpolicy` rules details](#rules-details).


## Data

Unlike the [default values](#default-values) and [rules](#rules), the Policy data is specific to each input `YAML` file. `genpolicy` generates Policy data that the [rules](#rules) can use to allow some of the Kata Agent [ttRPC API](../../libs/protocols/protos/agent.proto) requests depending on the input data of these requests. Any unexpected requests are rejected by the Policy.


# Rules details

This section provides details about the [rules](#rules) used by the Policy documents auto-generated by the `genpolicy` tool.

## `CopyFileRequest`

`CopyFile` requests are rejected by the auto-generated Policy unless the destination path of the file being copied matches at least one regular expression from `policy_data.request_defaults.CopyFileRequest`. By default, there is a single regex in `policy_data.request_defaults.CopyFileRequest`, copied by `genpolicy` from [`genpolicy-settings.json`](genpolicy-settings.json):

```
policy_data := {
    ...
    "request_defaults": {
        ...
        "CopyFileRequest": [
            "^$(cpath)/"
        ],
        ...
    },
    ...
}
```

The tool defines `$(cpath)` by copying its value from the same settings file into Policy's `policy_data.common.cpath`:

```
common := {
    ...
    "cpath": "/run/kata-containers/shared/containers",
    ...
}
```

Therefore, by default the auto-generated Policy allows the Host to copy any files under `/run/kata-containers/shared/containers` and rejects any other `CopyFile` requests. A user can alter this behavior by using a custom settings file including a different `policy_data.request_defaults.CopyFileRequest` field value, instead of using the default from `genpolicy-settings.json`.


## `CreateContainerRequest`

Most of the rules from [`rules.rego`](rules.rego) are applicable to the `CreateContainer` request, because:
1. The inputs of `CreateContainer` are very complex - e.g., see the Spec data structure from the [OCI protocol](../../libs/protocols/protos/oci.proto).
1. Those complex inputs could allow a buggy or malicious Host to alter the intended behavior of user's Kubernetes (K8s) pods.

For example, the Host could try to start in a confidential containers K8s pod a different container image than the image specified by user's `YAML`. Therefore, the Policy used by each pod has to verify that all the container images being used are exactly those that were referenced by the input `YAML` at the time when the Policy was created.

The auto-generated Policy data contains descriptions of the [`OCI Spec`](../../libs/protocols/protos/oci.proto) data structure corresponding to every container referenced by user's `YAML` file. For example, if `genpolicy` creates a Policy corresponding to a `Pod` that just starts a `busybox` shell, the tool will generate two `OCI` data structures in the Policy - one for the K8s `pause` container and another for the `busybox` shell. Example:

```
policy_data := {
    "containers": [
        {
            "OCI": {
                "Version": "1.1.0-rc.1",
                "Process": {
                    "Terminal": false,
                    "User": {
                        "UID": 65535,
                        "GID": 65535,
                        "AdditionalGids": [],
                        "Username": ""
                    },
                    "Args": [
                        "/pause"
                    ],
                    ...
                },
                ...
            },
            ...
        },
        {
            "OCI": {
                "Version": "1.1.0-rc.1",
                "Process": {
                    "Terminal": false,
                    "User": {
                        "UID": 0,
                        "GID": 0,
                        "AdditionalGids": [],
                        "Username": ""
                    },
                    "Args": [
                        "/bin/sh"
                    ],
                    ...
                },
                ...
            },
            ...
        }
    ],
    ...
```

The auto-generated Policy rules allow the creation of any container that matches at least one of the OCI Policy data structures.

**Warning** The auto-generated Policy doesn't keep track of which containers are already running in a pod. Therefore, in the example above the Kata Shim could start two shell containers instead of just one shell in a single pod - as long as both of these containers match the Policy data for user's shell container.

### [`OCI Spec`](../../libs/protocols/protos/oci.proto) validation

Following are examples of auto-generated Policy rules that check some of the `CreateContainer` input `OCI Spec` data structure fields:

1. The `Version` fields of the `OCI` Policy data and of the input `CreateContainer` data should match.
1. The container `OCI.Root.Readonly` field from the Policy and the input data should have the same value.
1. Each annotation of the container being created should match an annotation from the Policy data. **Warning** Container creation is allowed even if some of the Policy data annotations ***are not present*** in the input `OCI` data annotations. The auto-generated Policy just checks that those annotations that ***are present*** in the input `OCI` are allowed by the Policy data.
1. Verify that the values of the following annotations are consistent with the Policy data:
- `io.katacontainers.pkg.oci.bundle_path`
- `io.katacontainers.pkg.oci.container_type`
- `io.kubernetes.cri.container-name`
- `io.kubernetes.cri.container-type`
- `io.kubernetes.cri.sandbox-log-directory`
- `io.kubernetes.cri.sandbox-id`
- `io.kubernetes.cri.sandbox-name`
- `io.kubernetes.cri.sandbox-namespace`
- `nerdctl/network-namespace`
5. The input `OCI.Linux.Namespaces` information matches the Policy.
5. All the Policy `OCI.Linux.MaskedPaths` paths are present in the input `MaskedPaths` too. **Warning** The input `OCI.Linux.MaskedPaths` is allowed by the auto-generated Policy to include ***more*** paths than the Policy data. But, if a path is masked by the Policy's `oci.Linux.MaskedPaths`, a `CreateContainer` request is rejected if its input data doesn't mask the same path.
5. All the Policy `OCI.Linux.ReadonlyPaths` paths are present either in the input `ReadonlyPaths` or the input `MaskedPaths`. **Warning** the input `ReadonlyPaths` can contain ***more*** paths than the Policy `ReadonlyPaths`, but if the Policy designates a path as `Readonly` then that path must be designated either as `Readonly` or `Masked` by the `CreateContainer` input data.
5. The `Args`, `Cwd`, `NoNewPrivileges`, `Env` and other `OCI.Process` input field values are consistent with the Policy.
5. The input `OCI.Root.Path` matches the Policy data.
5. The input `OCI.Mounts` are allowed by Policy.

### `Storages` validation

`Storages` is another input field of Kata Agent's `CreateContainer` requests. The `Storages` Policy data for each container gets generated by `genpolicy` based on:
1. The container images referenced by user's `YAML` file.
1. Any `volumes` and `volumeMounts` information that might be present in user's `YAML` file.
1. The `volumes` data from [`genpolicy-settings.json`](genpolicy-settings.json).

Example of `Storages` data from an auto-generated Policy file:

```
policy_data := {
    "containers": [
        ...
        {
            "OCI": {
                ...
            },
            "storages": [
                {
                    "driver": "blk",
                    "driver_options": [],
                    "source": "",
                    "fstype": "tar",
                    "options": [
                        "$(hash0)"
                    ],
                    "mount_point": "$(layer0)",
                    "fs_group": null
                },
                {
                    "driver": "blk",
                    "driver_options": [],
                    "source": "",
                    "fstype": "tar",
                    "options": [
                        "$(hash1)"
                    ],
                    "mount_point": "$(layer1)",
                    "fs_group": null
                },
                {
                    "driver": "overlayfs",
                    "driver_options": [],
                    "source": "",
                    "fstype": "fuse3.kata-overlay",
                    "options": [
                        "2c342a137e693c7898aec36da1047f191dc7c1687e66198adacc439cf4adf379:2570e3a19e1bf20ddda45498a9627f61555d2d6c01479b9b76460b679b27d552",
                        "8568c70c0ccfe0051092e818da769111a59882cd19dd799d3bca5ffa82791080:b643b6217748983830b26ac14a35a3322dd528c00963eaadd91ef55f513dc73f"
                    ],
                    "mount_point": "$(cpath)/$(bundle-id)",
                    "fs_group": null
                },
                {
                    "driver": "local",
                    "driver_options": [],
                    "source": "local",
                    "fstype": "local",
                    "options": [
                        "mode=0777"
                    ],
                    "mount_point": "^$(cpath)/$(sandbox-id)/local/data$",
                    "fs_group": null
                },
                {
                    "driver": "ephemeral",
                    "driver_options": [],
                    "source": "tmpfs",
                    "fstype": "tmpfs",
                    "options": [],
                    "mount_point": "^/run/kata-containers/sandbox/ephemeral/data2$",
                    "fs_group": null
                }
            ],
            ...
        }
    ],
    ...
}
```

In this example, the corresponding `CreateContainer` request input is expected to include the following Kata Containers `Storages`:

1. Corresponding to container image layer 0.
1. Corresponding to container image layer 1.
1. Corresponding to the `overlay` of the container images.
1. For the `data` volume of the `YAML` example below.
1. For the `data2` volume of the `YAML` example below.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: persistent
spec:
...
  containers:
      ...
      volumeMounts:
        - mountPath: /busy1
          name: data
        - mountPath: /busy2
          name: data2
  volumes:
    - name: data
      emptyDir: {}
    - name: data2
      emptyDir:
        medium: Memory
```

`genpolicy` auto-generates the Policy `overlay` layer storage data structure. That structure provides some of the information used to validate the integrity of each container image referenced by user's `YAML` file:
1. An ordered collection of layer IDs.
1. For each layer ID, a `dm-verity` root hash value.

Each container image layer is exposed by the Kata Shim to the Guest VM as a `dm-verity` protected block storage device. If the `CreateContainer` input layer IDs and `dm-verity` root hashes match those from the Policy:
1. The Kata Agent uses the IDs and root hashes and to mount the container image layer storage devices.
1. The Guest kernel ensures the integrity of the container image, by checking the `dm-verity` information of each layer.

## `ExecProcessRequest`

`ExecProcess` requests are rejected by the auto-generated Policy unless:
1. They correspond to an `exec` K8s `livenessProbe`, `readinessProbe` or `startupProbe`, or
1. They correspond to the `policy_data.request_defaults.ExecProcessRequest` data from Policy.

### `exec` probes

Given this example `genpolicy` input `YAML` file as input:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: exec-test
spec:
  containers:
    ...
    - command:
        - /bin/sh
      env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: status.podIP
      readinessProbe:
        exec:
          command:
            - echo
            - "Ready ${POD_IP}!"
        failureThreshold: 1
        periodSeconds: 5
        timeoutSeconds: 10
```

the tool generates the following `ExecProcessRequest` Policy data:

```
policy_data := {
    "containers": [
        ...
        {
            "OCI": {
                ...
            },
            "storages": [
                ...
            ],
            "exec_commands": [
                "echo Ready ${POD_IP}!",
            ]
        }
    ]
}
```

### `policy_data.request_defaults.ExecProcessRequest`

An `ExecProcess` request is allowed by the auto-generated Policy if its command line matches at least one entry from the `commands` and/or the `regex` fields of the `policy_data.request_defaults.ExecProcessRequest` data structure.

The `commands` and the `regex` entries get copied by `genpolicy` from [`genpolicy-settings.json`](genpolicy-settings.json). By default there are no such entries (both the `commands` and the `regex` collections are empty), so no `ExecProcess` requests are allowed by these two collections.

A user that wants to allow some of the `ExecProcess` requests can specify a modified copy of `genpolicy-settings.json` as parameter to `genpolicy`.

**Warning** The `commands` are easier to use, but require specifying the full command line being allowed by the Policy. The `regex` entries are more flexible - because a single entry can allow multiple `ExecProcess` command lines - but are easier to misuse e.g., by users that are not regular expression experts.

Examples of `policy_data.request_defaults.ExecProcessRequest.commands` entries:

```
policy_data := {
    ...
    "request_defaults": {
        ...
        "ExecProcessRequest": {
            "commands": [
                "/bin/bash",
                "/bin/myapp -p1 -p2"
            ],
            "regex": []
        },
        ...
    }
}
```

Examples of `policy_data.request_defaults.ExecProcessRequest.regex` entries:

```
policy_data := {
    ...
    "request_defaults": {
        ...
        "ExecProcessRequest": {
            "commands": [],
            "regex": [
                "^/bin/sh -x -c echo hostName \\| nc -v -t -w 2 externalname-service [0-9]+$",
                "^/bin/sh -x -c echo hostName \\| nc -v -t -w 2 [0-9]+\\.[0-9]+\\.[0-9]+\\.[0-9]+ [0-9]+$"
            ]
        },
        ...
    }
}
```

## `ReadStreamRequest`

`ReadStream` requests are rejected by the default auto-generated Policy. A user can allow the Kata Containers Shim to read the `stdout`/`stderr` streams of the Guest VM containers by allowing these requests using a modified [`genpolicy-settings.json`](genpolicy-settings.json) - e.g.,

```
policy_data := {
    ...
    "request_defaults": {
        ...
       "ReadStreamRequest": true,
        ...
    }
}
```

## `WriteStreamRequest`

By default, `WriteStream` requests are rejected by the auto-generated Policy. A user can allow the Kata Containers Shim to send input to the `stdin` of Guest VM containers by allowing these requests using a modified [`genpolicy-settings.json`](genpolicy-settings.json) - e.g.,

```
policy_data := {
    ...
    "request_defaults": {
        ...
       "WriteStreamRequest": true,
        ...
    }
}
```