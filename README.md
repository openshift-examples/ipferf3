---
title: iperf3 example
linktitle: iperf3
description: Network perf analyse via iperf3
tags:
  - iperf3
---

# Network Performance analyze via iperf3

Start daemon sets

```
oc new-project iperf-test
oc apply -f iperf.daemonset.yaml
oc create sa hostnetwork
oc adm policy add-scc-to-user hostnetwork-v2 -z hostnetwork
oc apply -f iperf-hostnetwork.daemonset.yaml
```
Run network performance tests via:

```bash
# Example ucs56 -> ucs57

$ oc get pods -o wide | grep ucs
iperf-frnhr               1/1     Running   0          4d2h    10.129.11.203   ucs56    <none>           <none>
iperf-hostnetwork-5h8t6   1/1     Running   0          4d1h    10.32.96.57     ucs57    <none>           <none>
iperf-hostnetwork-tzvlp   1/1     Running   0          4d1h    10.32.96.56     ucs56    <none>           <none>
iperf-m9x2f               1/1     Running   0          4d2h    10.130.9.190    ucs57    <none>           <none>

# via Pod Network
$ oc rsh iperf-frnhr
sh-4.4$ iperf3 --client  10.130.9.190   --time 60 --parallel 4
Connecting to host 10.130.9.190, port 5201
....
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-60.00  sec  14.4 GBytes  2.06 Gbits/sec  13119             sender
[  5]   0.00-60.04  sec  14.4 GBytes  2.05 Gbits/sec                  receiver
[  7]   0.00-60.00  sec  14.3 GBytes  2.05 Gbits/sec  12867             sender
[  7]   0.00-60.04  sec  14.3 GBytes  2.05 Gbits/sec                  receiver
[  9]   0.00-60.00  sec  14.4 GBytes  2.06 Gbits/sec  12260             sender
[  9]   0.00-60.04  sec  14.4 GBytes  2.05 Gbits/sec                  receiver
[ 11]   0.00-60.00  sec  14.0 GBytes  2.00 Gbits/sec  13686             sender
[ 11]   0.00-60.04  sec  14.0 GBytes  2.00 Gbits/sec                  receiver
[SUM]   0.00-60.00  sec  57.0 GBytes  8.16 Gbits/sec  51932             sender
[SUM]   0.00-60.04  sec  57.0 GBytes  8.16 Gbits/sec                  receiver

iperf Done.

# via Host Network

$ oc rsh iperf-hostnetwork-tzvlp
sh-4.4$ iperf3 --client  10.32.96.57    --time 60 --parallel 4
Connecting to host 10.32.96.57, port 5201
....
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-60.00  sec  14.8 GBytes  2.12 Gbits/sec  2200             sender
[  5]   0.00-60.04  sec  14.8 GBytes  2.11 Gbits/sec                  receiver
[  7]   0.00-60.00  sec  16.1 GBytes  2.31 Gbits/sec  2330             sender
[  7]   0.00-60.04  sec  16.1 GBytes  2.30 Gbits/sec                  receiver
[  9]   0.00-60.00  sec  15.6 GBytes  2.23 Gbits/sec  2612             sender
[  9]   0.00-60.04  sec  15.6 GBytes  2.23 Gbits/sec                  receiver
[ 11]   0.00-60.00  sec  18.4 GBytes  2.63 Gbits/sec  2547             sender
[ 11]   0.00-60.04  sec  18.4 GBytes  2.63 Gbits/sec                  receiver
[SUM]   0.00-60.00  sec  64.9 GBytes  9.29 Gbits/sec  9689             sender
[SUM]   0.00-60.04  sec  64.9 GBytes  9.28 Gbits/sec                  receiver

iperf Done.

# via Pod Network & Service

$ oc label pod/iperf-m9x2f host=ucs57
$ oc create -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: target
spec:
  ports:
  - name: 5201-5201
    port: 5201
    protocol: TCP
    targetPort: 5201
  selector:
    host: ucs57
EOF
$ oc get svc
NAME     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
target   ClusterIP   172.30.45.37   <none>        5201/TCP   4s
$ oc rsh iperf-frnhr
sh-4.4$ iperf3 --client  172.30.45.37    --time 60 --parallel 4
Connecting to host 172.30.45.37, port 5201
....
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-60.00  sec  14.5 GBytes  2.08 Gbits/sec  10836             sender
[  5]   0.00-60.04  sec  14.5 GBytes  2.08 Gbits/sec                  receiver
[  7]   0.00-60.00  sec  14.6 GBytes  2.08 Gbits/sec  11719             sender
[  7]   0.00-60.04  sec  14.6 GBytes  2.08 Gbits/sec                  receiver
[  9]   0.00-60.00  sec  14.2 GBytes  2.04 Gbits/sec  12327             sender
[  9]   0.00-60.04  sec  14.2 GBytes  2.04 Gbits/sec                  receiver
[ 11]   0.00-60.00  sec  14.7 GBytes  2.11 Gbits/sec  11120             sender
[ 11]   0.00-60.04  sec  14.7 GBytes  2.11 Gbits/sec                  receiver
[SUM]   0.00-60.00  sec  58.1 GBytes  8.31 Gbits/sec  46002             sender
[SUM]   0.00-60.04  sec  58.1 GBytes  8.31 Gbits/sec                  receiver

iperf Done.
```

## Containerfile

```
FROM quay.io/dfroehli/tools:latest
EXPOSE 5201/tcp
EXPOSE 5201/udp
CMD iperf3 $IPERF_OPT
```

```
FROM registry.access.redhat.com/ubi8/ubi
RUN  yum -y install net-tools bind-utils iputils less strace tcpdump nmap-ncat iperf3 vim-minimal socat hostname && \
     yum clean all
```


