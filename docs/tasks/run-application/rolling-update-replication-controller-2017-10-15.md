---
approvers:
- janetkuo
title: 使用Relication Controller进行滚动更新
---
<!-- 
---
approvers:
- janetkuo
title: Perform Rolling Update Using a Replication Controller
--- 
-->

* TOC
{:toc}

<!-- ## Overview -->
## 概览

<!-- 
**Note**: The preferred way to create a replicated application is to use a
[Deployment](/docs/api-reference/{{page.version}}/#deployment-v1beta1-apps),
which in turn uses a
[ReplicaSet](/docs/api-reference/{{page.version}}/#replicaset-v1beta1-extensions).
For more information, see
[Running a Stateless Application Using a Deployment](/docs/tasks/run-application/run-stateless-application-deployment/). 
-->
**注意**: 创建副本应用的推荐方法是使用
[Deployment](/docs/api-reference/{{page.version}}/#deployment-v1beta1-apps)
并配置[ReplicaSet](/docs/api-reference/{{page.version}}/#replicaset-v1beta1-extensions)。
详细信息请参考
[Running a Stateless Application Using a Deployment](/docs/tasks/run-application/run-stateless-application-deployment/)。

<!-- To update a service without an outage, `kubectl` supports what is called [rolling update](/docs/user-guide/kubectl/{{page.version}}/#rolling-update), which updates one pod at a time, rather than taking down the entire service at the same time. See the [rolling update design document](https://git.k8s.io/community/contributors/design-proposals/cli/simple-rolling-update.md) and the [example of rolling update](/docs/tasks/run-application/rolling-update-replication-controller/) for more information. -->
为了实现不停机升级服务，`kubectl` 支持 [rolling-update](/docs/user-guide/kubectl/{{page.version}}/#rolling-update)功能，通过逐个更新pod的方式避免（同时更新所有pod而导致）整个服务下线。详情请参考 [rolling-update设计文档](https://git.k8s.io/community/contributors/design-proposals/cli/simple-rolling-update.md)以及[rolling-update示例](/docs/tasks/run-application/rolling-update-replication-controller/)。

<!-- 
Note that `kubectl rolling-update` only supports Replication Controllers. However, if you deploy applications with Replication Controllers,
consider switching them to [Deployments](/docs/concepts/workloads/controllers/deployment/). A Deployment is a higher-level controller that automates rolling updates
of applications declaratively, and therefore is recommended. If you still want to keep your Replication Controllers and use `kubectl rolling-update`, keep reading: 
-->
请注意 `kubectl rolling-update` 仅支持Replication Controllers。但是如果您通过Replication Controllers来部署应用的话，请考虑转为使用[Deployments](/docs/concepts/workloads/controllers/deployment/)。Deployment是更高层次的控制组件，推荐用于需要自动化滚动更新的声明式应用。假如您仍然希望保留Replication Controllers 并使用 `kubectl rolling-update` ，请继续阅读下文：
<!-- 
A rolling update applies changes to the configuration of pods being managed by
a replication controller. The changes can be passed as a new replication
controller configuration file; or, if only updating the image, a new container
image can be specified directly. 
-->
当使用滚动更新对Replication Controller管理的pod的配置生效变更时，配置变更可以通过传入
一个新的Replication Controller配置文件来实现，或者假如只更新镜像的话可以直接指定一个新
的容器镜像。

A rolling update works by:
<!-- 
1. Creating a new replication controller with the updated configuration.
2. Increasing/decreasing the replica count on the new and old controllers until
   the correct number of replicas is reached.
3. Deleting the original replication controller. 
-->
一次滚动更新包含以下步骤：

1. 使用更新的配置创建一个新replication controller。
2. 在新旧RC上增/减副本数量至更新配置中定义的副本数。
3. 删除旧replication controller。

<!-- 
Rolling updates are initiated with the `kubectl rolling-update` command: 
-->
滚动更新使用 `kubectl rolling-update` 命令启动：

    $ kubectl rolling-update NAME \
        ([NEW_NAME] --image=IMAGE | -f FILE)

<!-- 
## Passing a configuration file 
-->
## 传入配置文件

<!-- 
To initiate a rolling update using a configuration file, pass the new file to 
-->
如需使用配置文件来启动滚动更新，可以通过
`kubectl rolling-update` 传入新文件:

    $ kubectl rolling-update NAME -f FILE

<!-- 
The configuration file must:

* Specify a different `metadata.name` value.

* Overwrite at least one common label in its `spec.selector` field.

* Use the same `metadata.namespace`.

Replication controller configuration files are described in
[Creating Replication Controllers](/docs/tutorials/stateless-application/run-stateless-ap-replication-controller/). 
-->
配置文件必须：

* 指定一个不同的 `metadata.name` 值。

* 在 `spec.selector` 字段内覆盖至少一个通用标签（common label）

* 使用相同的 `metadata.namespace` 。

Replication controller配置文件请参考
[Creating Replication Controllers](/docs/tutorials/stateless-application/run-stateless-ap-replication-controller/)

<!-- 
### Examples

    // Update pods of frontend-v1 using new replication controller data in frontend-v2.json.
    $ kubectl rolling-update frontend-v1 -f frontend-v2.json

    // Update pods of frontend-v1 using JSON data passed into stdin.
    $ cat frontend-v2.json | kubectl rolling-update frontend-v1 -f - 
-->
### 示例

    // 使用frontend-v2.json内的新replication controller数据更新frontend-v1的pod.
    $ kubectl rolling-update frontend-v1 -f frontend-v2.json

    // 使用传入stdin的JSON数据更新frontend-v1的pod.
    $ cat frontend-v2.json | kubectl rolling-update frontend-v1 -f -
<!-- 
## Updating the container image 
-->
## 更新容器镜像

<!-- 
To update only the container image, pass a new image name and tag with the
`--image` flag and (optionally) a new controller name: 
-->
只更新容器镜像的话，可以通过 `--image` 参数标识（flag）和新RC名称（可选项）传入新镜像名称和标签（tag）：

    $ kubectl rolling-update NAME [NEW_NAME] --image=IMAGE:TAG
<!-- 
The `--image` flag is only supported for single-container pods. Specifying
`--image` with multi-container pods returns an error. 
-->
 `--image` 参数标识只支持单容器pod。对多容器pod使用`--image` 将返回出错信息。

<!-- 
If no `NEW_NAME` is specified, a new replication controller is created with
a temporary name. Once the rollout is complete, the old controller is deleted,
and the new controller is updated to use the original name. 
-->
假如未指定 `NEW_NAME` ，将使用一个临时名称来创建新replication controller。一旦更新完成旧RC将被删除，而新RC将更新为使用原来的名称。

<!-- 
The update will fail if `IMAGE:TAG` is identical to the
current value. For this reason, we recommend the use of versioned tags as
opposed to values such as `:latest`. Doing a rolling update from `image:latest`
to a new `image:latest` will fail, even if the image at that tag has changed.
Moreover, the use of `:latest` is not recommended, see
[Best Practices for Configuration](/docs/concepts/configuration/overview/#container-images) for more information. 
-->
假如 `IMAGE:TAG` 与当前值相同滚动更新将会失败。因此我们建议使用带版本号的标签（tag）而不是使用类似于 `:latest` 这样的标签值。当从 `image:latest` 滚动更新到一个新的 `image:latest`  时，即使镜像发生了变化也还是会失败。因此不推荐使用 `:latest` 。更多信息请参考[Best Practices for Configuration](/docs/concepts/configuration/overview/#container-images)。

<!--
 ### Examples

    // Update the pods of frontend-v1 to frontend-v2
    $ kubectl rolling-update frontend-v1 frontend-v2 --image=image:v2

    // Update the pods of frontend, keeping the replication controller name
    $ kubectl rolling-update frontend --image=image:v2 
-->
### 示例

    // 将所有名称为frontend-v1的pod更新为frontend-v2
    $ kubectl rolling-update frontend-v1 frontend-v2 --image=image:v2

    // 更新所有名称为frontend的pods, 保持原有replication controller名称不变
    $ kubectl rolling-update frontend --image=image:v2

<!-- 
## Required and optional fields 
-->
## 必填及可选字段

<!-- 
Required fields are:

* `NAME`: The name of the replication controller to update.

as well as either:

* `-f FILE`: A replication controller configuration file, in either JSON or
  YAML format. The configuration file must specify a new top-level `id` value
  and include at least one of the existing `spec.selector` key:value pairs.
  See the
  [Run Stateless AP Replication Controller](/docs/tutorials/stateless-application/run-stateless-ap-replication-controller/#replication-controller-configuration-file)
  page for details.
<br>
<br>
    or:
<br>
<br>
* `--image IMAGE:TAG`: The name and tag of the image to update to. Must be
  different than the current image:tag currently specified. 
-->
必填字段包括：

* `NAME`: 需要更新的replication controller名称

以及

* `-f FILE`: 一个JSON或YAML格式的replication controller配置文件。配置文件必须指定一个新的 top-level `id` 值，并包含至少一个现有的 `spec.selector` key:value 值。
  详情请参考
  [Run Stateless AP Replication Controller](/docs/tutorials/stateless-application/run-stateless-ap-replication-controller/#replication-controller-configuration-file).
<br>
<br>
    或:
<br>
<br>
* `--image IMAGE:TAG`: 需要更新的新镜像的image:tag值，不能和当前的image:tag相同。

<!--
 Optional fields are:

* `NEW_NAME`: Only used in conjunction with `--image` (not with `-f FILE`). The
  name to assign to the new replication controller.
* `--poll-interval DURATION`: The time between polling the controller status
  after update. Valid units are `ns` (nanoseconds), `us` or `µs` (microseconds),
  `ms` (milliseconds), `s` (seconds), `m` (minutes), or `h` (hours). Units can
  be combined (e.g. `1m30s`). The default is `3s`.
* `--timeout DURATION`: The maximum time to wait for the controller to update a
  pod before exiting. Default is `5m0s`. Valid units are as described for
  `--poll-interval` above.
* `--update-period DURATION`: The time to wait between updating pods. Default
  is `1m0s`. Valid units are as described for `--poll-interval` above. 
  -->
可选字段包括：

* `NEW_NAME`: 只能与 `--image` 组合使用(不可用于 `-f FILE`)，用于为新的replication controller指定命名。
* `--poll-interval DURATION`: 滚动更新后轮询RC状态的间隔时间。 可用的单位有 `ns` (纳秒), `us` 或 `µs` (微秒),
  `ms` (毫秒), `s` (秒), `m` (分钟), or `h` (小时). 单位可组合使用 (例如 `1m30s`). 默认值是 `3s`.
* `--timeout DURATION`: RC更新pod的最长等待时间，超时将会退出更新。 默认为 `5m0s`. 可用的单位与前面所述的
  `--poll-interval` 相同。
* `--update-period DURATION`: 更新pod时的等待时间. 默认为 `1m0s`。可用的单位与前面所述的
  `--poll-interval` 相同。

<!-- 
Additional information about the `kubectl rolling-update` command is available
from the [`kubectl` reference](/docs/user-guide/kubectl/{{page.version}}/#rolling-update). 
-->
关于 `kubectl rolling-update` 命令的额外信息可参考[`kubectl` reference](/docs/user-guide/kubectl/{{page.version}}/#rolling-update)。

<!-- 
## Walkthrough

Let's say you were running version 1.7.9 of nginx: 
-->
## 操作实例

假如你已经运行着nginx的1.7.9版本：

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: my-nginx
spec:
  replicas: 5
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

<!--
 To update to version 1.9.1, you can use [`kubectl rolling-update --image`](https://git.k8s.io/community/contributors/design-proposals/cli/simple-rolling-update.md) to specify the new image: 
 -->
要更新到1.9.1版本，你可以使用 [`kubectl rolling-update --image`]
(https://git.k8s.io/community/contributors/design-proposals/cli/simple-rolling-update.md) 来指定新镜像：

```shell
$ kubectl rolling-update my-nginx --image=nginx:1.9.1
Created my-nginx-ccba8fbd8cc8160970f63f9a2696fc46
```
<!-- 
In another window, you can see that `kubectl` added a `deployment` label to the pods, whose value is a hash of the configuration, to distinguish the new pods from the old: 
-->
在另外一个窗口你可以看到 `kubectl` 为pod增加了一个 `deployment` 标签，内容为配置的哈希值（hash），用于区分新旧pod。

```shell
$ kubectl get pods -l app=nginx -L deployment
NAME                                              READY     STATUS    RESTARTS   AGE       DEPLOYMENT
my-nginx-ccba8fbd8cc8160970f63f9a2696fc46-k156z   1/1       Running   0          1m        ccba8fbd8cc8160970f63f9a2696fc46
my-nginx-ccba8fbd8cc8160970f63f9a2696fc46-v95yh   1/1       Running   0          35s       ccba8fbd8cc8160970f63f9a2696fc46
my-nginx-divi2                                    1/1       Running   0          2h        2d1d7a8f682934a254002b56404b813e
my-nginx-o0ef1                                    1/1       Running   0          2h        2d1d7a8f682934a254002b56404b813e
my-nginx-q6all                                    1/1       Running   0          8m        2d1d7a8f682934a254002b56404b813e
```

<!--
 `kubectl rolling-update` reports progress as it progresses: 
 -->
`kubectl rolling-update` 会随着运行过程显示执行的进度：

```
Scaling up my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 from 0 to 3, scaling down my-nginx from 3 to 0 (keep 3 pods available, don't exceed 4 pods)
Scaling my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 up to 1
Scaling my-nginx down to 2
Scaling my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 up to 2
Scaling my-nginx down to 1
Scaling my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 up to 3
Scaling my-nginx down to 0
Update succeeded. Deleting old controller: my-nginx
Renaming my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 to my-nginx
replicationcontroller "my-nginx" rolling updated
```

<!-- 
If you encounter a problem, you can stop the rolling update midway and revert to the previous version using `--rollback`: 
-->
假如遇到问题，可以使用 `--rollback` 命令中途停止滚动升级并回退到之前的版本：

```shell
$ kubectl rolling-update my-nginx --rollback
Setting "my-nginx" replicas to 1
Continuing update with existing controller my-nginx.
Scaling up nginx from 1 to 1, scaling down my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 from 1 to 0 (keep 1 pods available, don't exceed 2 pods)
Scaling my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 down to 0
Update succeeded. Deleting my-nginx-ccba8fbd8cc8160970f63f9a2696fc46
replicationcontroller "my-nginx" rolling updated
```
<!-- 
This is one example where the immutability of containers is a huge asset. 
-->
这是不可变容器是巨型资产的一个例子。

If you need to update more than just the image (e.g., command arguments, environment variables), you can create a new replication controller, with a new name and distinguishing label value, such as:

假如您要更新的不仅仅是镜像（比如命令参数、环境变量），可以新创建一个带有新名称和新标识lable值的RC，像下面这样：

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: my-nginx-v4
spec:
  replicas: 5
  selector:
    app: nginx
    deployment: v4
  template:
    metadata:
      labels:
        app: nginx
        deployment: v4
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.2
        args: ["nginx", "-T"]
        ports:
        - containerPort: 80
```

and roll it out:

然后进行滚动升级：

```shell
$ kubectl rolling-update my-nginx -f ./nginx-rc.yaml
Created my-nginx-v4
Scaling up my-nginx-v4 from 0 to 5, scaling down my-nginx from 4 to 0 (keep 4 pods available, don't exceed 5 pods)
Scaling my-nginx-v4 up to 1
Scaling my-nginx down to 3
Scaling my-nginx-v4 up to 2
Scaling my-nginx down to 2
Scaling my-nginx-v4 up to 3
Scaling my-nginx down to 1
Scaling my-nginx-v4 up to 4
Scaling my-nginx down to 0
Scaling my-nginx-v4 up to 5
Update succeeded. Deleting old controller: my-nginx
replicationcontroller "my-nginx-v4" rolling updated
```

## Troubleshooting

If the `timeout` duration is reached during a rolling update, the operation will
fail with some pods belonging to the new replication controller, and some to the
original controller.

To continue the update from where it failed, retry using the same command.

To roll back to the original state before the attempted update, append the
`--rollback=true` flag to the original command. This will revert all changes.
