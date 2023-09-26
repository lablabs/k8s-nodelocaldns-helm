# NodeLocal DNSCache Helm chart

## Helm Chart Repository

```
helm repo add cloudflare-exporter https://lablabs.github.io/k8s-nodelocaldns-helm/
helm install k8s-nodelocaldns-helm/node-local-dns
```

## Configuration

This chart deploys NodeLocal DNSCache Daemon set according to https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/.

It is designed to work both with iptables and IPVS setup.

Latest available `node-local-dns` image can be found at [node-local-dns google container repository](https://console.cloud.google.com/gcr/images/google-containers/GLOBAL/k8s-dns-node-cache)

### Values

The following table lists the configurable parameters of the Node-local-dns chart and their default values.

| Parameter                | Description             | Default        |
| ------------------------ | ----------------------- | -------------- |
| `image.repository` |  | `"k8s.gcr.io/dns/k8s-dns-node-cache"` |
| `image.pullPolicy` |  | `"IfNotPresent"` |
| `image.tag` |  | `"1.22.9"` |
| `image.args.skipTeardown` |  | `true` |
| `image.args.syncInterval` |  | `"1ns"` |
| `image.args.interfaceName` |  | `"nodelocaldns"` |
| `image.args.upstreamSvc` |  | `"kube-dns"` |
| `image.args.healthPort` | `8080` |
| `image.args.setupIptables` | `false` |
| `image.args.setupEbtables` | `false` |
| `image.args.quiet` | `false` |
| `imagePullSecrets` |  | `[]` |
| `config.localDnsIp` |  | `"169.254.20.11"` |
| `config.zones` |  | `[{".:53":{"plugins":{"errors":true,"reload":true,"debug":false,"log":{"format":"combined","classes":"all"},"cache":{"parameters":30,"denial":{},"success":{},"prefetch":{},"serve_stale":false},"forward":{"parameters":"__PILLAR__UPSTREAM__SERVERS__","force_tcp":false,"prefer_udp":false,"policy":"","max_fails":"","expire":"","health_check":"","except":""},"prometheus":true,"health":{"port":8080}}}},{"ip6.arpa:53":{"plugins":{"errors":true,"reload":true,"debug":false,"log":{"format":"combined","classes":"all"},"cache":{"parameters":30},"forward":{"parameters":"__PILLAR__UPSTREAM__SERVERS__","force_tcp":false},"prometheus":true,"health":{"port":8080}}}},{"in-addr.arpa:53":{"plugins":{"errors":true,"reload":true,"debug":false,"log":{"format":"combined","classes":"all"},"cache":{"parameters":30},"forward":{"parameters":"__PILLAR__UPSTREAM__SERVERS__","force_tcp":false},"prometheus":true,"health":{"port":8080}}}}]` |
| `useHostNetwork` |  | `true` |
| `updateStrategy.rollingUpdate.maxUnavailable` |  | `"10%"` |
| `priorityClassName` |  | `"system-node-critical"` |
| `podAnnotations` |  | `{}` |
| `podSecurityContext` |  | `{}` |
| `securityContext.privileged` |  | `true` |
| `readinessProbe` |  | `null` |
| `serviceAccount.create` |  | `true` |
| `serviceAccount.annotations` |  | `{}` |
| `serviceAccount.name` |  | `""` |
| `nodeSelector` |  | `{}` |
| `affinity` |  | `{}` |
| `tolerations` |  | `[{"key": "CriticalAddonsOnly", "operator": "Exists"}, {"effect": "NoExecute", "operator": "Exists"}, {"effect": "NoSchedule", "operator": "Exists"}]` |
| `resources.requests.cpu` |  | `"30m"` |
| `resources.requests.memory` |  | `"50Mi"` |
| `metrics.prometheusScrape` |  | `"true"` |
| `metrics.port` |  | `9253` |
| `podmonitor.metricRelabelings` | `Define metric relabeling configurations for Prometheus` | [] |

### Setting upstream DNS service

```yaml
config:
  localDnsIp: 169.254.20.11
  zones:
    - .:53:
        plugins:
          errors: true
          reload: true
          debug: false
          log:
            fomat: common
            classes: all
          cache:
            parameters: 30
            denial: {}
              # size: 0
              # ttl: 1
            success: {}
              # size: 8192
              # ttl: 30
            prefetch: {}
              # amount: 1
              # duration: 10m
              # percentage: 20%
            serve_stale: false
          forward:
            parameters: __PILLAR__UPSTREAM__SERVERS__ # defaults to /etc/resolv.conf
            force_tcp: false
            prefer_udp: false
            policy: "" # random|round_robin|sequential
            max_fails: "" # 10
            expire: "" # 10s
            health_check: "" # 10s
          prometheus: true
          health:
            port: 8080
```

```yaml
  Corefile: |
    __PILLAR__DNS__DOMAIN__:53 {
        errors
        cache {
          success 9984 30
          denial 9984 5
        }
        reload
        loop
        bind __PILLAR__LOCAL__DNS__ __PILLAR__DNS__SERVER__
        forward . __PILLAR__CLUSTER__DNS__ {
          force_tcp
        }
        prometheus :9253
        health __PILLAR__LOCAL__DNS__:8080
    }
    in-addr.arpa:53 {
        errors
        cache 30
        reload
        loop
        bind __PILLAR__LOCAL__DNS__ __PILLAR__DNS__SERVER__
        forward . __PILLAR__CLUSTER__DNS__ {
          force_tcp
        }
        prometheus :9253
    }
    ip6.arpa:53 {
        errors
        cache 30
        reload
        loop
        bind __PILLAR__LOCAL__DNS__ __PILLAR__DNS__SERVER__
        forward . __PILLAR__CLUSTER__DNS__ {
          force_tcp
        }
        prometheus :9253
    }
    .:53 {
        errors
        cache 30
        reload
        loop
        bind __PILLAR__LOCAL__DNS__ __PILLAR__DNS__SERVER__
        forward . __PILLAR__UPSTREAM__SERVERS__
        prometheus :9253
    }
```

https://github.com/kubernetes/kubernetes/blob/master/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml