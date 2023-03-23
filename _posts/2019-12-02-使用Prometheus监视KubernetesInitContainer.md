---
title: 使用Prometheus监视KubernetesInitContainer
date: 2019-12-02 12:23
tag: kubernetes
---
Kubernetes InitContainers是在容器启动之前运行任意代码的一种简便方法。它可以确保在启动和运行应用程序之前满足某些前提条件。

但是InitContainers 也有其局限性，如果Crashloop永远不会重新运行InitContainers，InitContainers仅在pod创建时运行，如果pod中的常规容器死亡并重新启动，则只需重新启动它们即可。不需要运行init容器，因为emptydir卷与pod共享相同的生命周期，只要pod继续存在。

这个时候对InitContainer的监控报警非常有必要，可以在排查Pod异常的时候提供前期的信息，而不是像之前一样是一个黑盒，Kube-state-metrics为Prometheus公开了大量Kubernetes集群指标。结合两者，我们可以在发现容器问题时进行监视和警报。社区合并了一个提供InitContainer数据的Metrics指标[PR](https://github.com/kubernetes/kube-state-metrics/pull/762)。

> Kube-state-metrics 在1.7版本提供了metrics`kube_pod_init_container_status_last_terminated_reason`
>
> https://github.com/kubernetes/kube-state-metrics/releases/tag/v1.7.0
>
> [FEATURE] Add Pod init container metrics. [#762](https://github.com/kubernetes/kube-state-metrics/pull/762)

添加相关指标到Prometheus的`scrape_configs`

```
- job_name: 'kube-state-metrics'
  static_configs:
    - targets: ['kube-state-metrics:8080']
```

`kube_pod_init_container_status_last_terminated_reason`包含`reason`可以处于五个不同状态的度量标准标签：

- Completed
- OOMKilled
- Error
- ContainerCannotRun
- DeadlineExceeded

如果需要将这些监控指标不是"Completed"被AlertManager的报出来，需要添加 如下Alert告警规则

```yaml
groups:
  - name: Init container failure
    rules:
      - alert: InitContainersFailed
        expr: kube_pod_init_container_status_last_terminated_reason{reason!="Completed"} == 1
        annotations:
          summary: '{{ $labels.container }} init failed'
          description: '{{ $labels.container }} has not completed init containers with the reason {{ $labels.reason }}'
```

