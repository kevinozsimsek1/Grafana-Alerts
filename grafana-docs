# Grafana Alert Kuralları

Kaynak: Grafana Alerting YAML export (provisioning format, apiVersion: 1, orgId: 3). Alert-rules klasörü altında 4 grup var: 5xx Root Cause, eks-prod-cluster, Endpoint Latency, RDS.

| Grup | Durum | Interval |
|---|---|---|
| 5xx Root Cause | 2 firing, 2 no data, 9 normal | 1m |
| eks-prod-cluster | 1 firing, 18 normal | 1m |
| Endpoint Latency | 2 normal | 1m |
| RDS | belirtilmedi | 1m |

## 1. 5xx Root Cause

Dashboard: 5xx-root-cause-analysis-v9. Toplam kural: 11 (2 firing, 2 no data, 9 normal)

| Kural | Katman | Koşul | For | Severity | Receiver |
|---|---|---|---|---|---|
| 5XX RCA - Authorizer Failures - Warning | API Gateway | Authorizer 500 hataları > 0 / 5m | 5m | warning | — |
| 5XX RCA - Authorizer Failures - Critical | API Gateway | Authorizer 500 hataları > 5 / 2m | 2m | critical | — |
| ALB Error Rates 5xx | Application Load Balancer | HTTPCode_ELB_5XX_Count > 0 | 1m | critical | — |
| 5XX RCA - Integration Failures - Warning | API Gateway Backend Integration | Integration hataları > 0 / 5m | 5m | warning | — |
| 5XX RCA - Integration Failures - Critical | API Gateway Backend Integration | Integration hataları > 5 / 2m | 2m | critical | — |
| 5XX RCA - API GW 5XX - Warning | API Gateway (tüm API'ler) | 5XXError metriği > 1 / 5m | 5m | warning | — |
| 5XX RCA - API GW 5XX - Critical | API Gateway (tüm API'ler) | 5XXError metriği > 10 / 5m | 2m | critical | — |
| 5XX RCA - .NET OTel 5XX - Warning | .NET Servisleri | 5xx yanıt sayısı > 1 / 5m | 5m | warning | — |
| 5XX RCA - .NET OTel 5XX - Critical | .NET Servisleri | 5xx yanıt sayısı > 10 / 5m | 2m | critical | — |
| Top Endpoints Hata Warning | API Gateway (endpoint bazlı) | Hata sayısı 3.5-9.5 aralığında | 5m | warning | — |
| Top Endpoints Hata Critical Alert | API Gateway (endpoint bazlı) | Hata sayısı > 9.5 | 5m | critical | SRE System Alerts Critical |

Mantık: 5xx hatalarını katmanlara ayırarak (Authorizer, Integration, genel API GW, .NET servis seviyesi) sorunun nerede olduğunu ayırt etmek.

No data nedeni: Authorizer/Integration kuralları CloudWatch Logs Insights sorgusuna dayanıyor. Sorgu o periyotta log bulamazsa noDataState:NoData nedeniyle "no data" görünür — genelde hata olmadığı anlamına gelir, kesinti değil.

## 2. eks-prod-cluster

Cluster: eks-prod, team: sre. Toplam kural: 19 (1 firing, 18 normal)

| Kural | Koşul | For | Severity | Receiver |
|---|---|---|---|---|
| EKS PROD - Node NotReady | En az bir node Ready değil | 5m | critical | SRE System Alerts Critical |
| Deployment Down - Zero Available Replicas | spec.replicas>0 iken available=0 olan deployment sayısı > 0.5 | 1m | critical | — |
| EKS PROD - CrashLoopBackOff | CrashLoopBackOff container sayısı > 0 | 5m | critical | — |
| EKS PROD - Deployment Unhealthy | Available=false (5dk kesintisiz) deployment sayısı > 0 | 1m | critical | — |
| EKS PROD - ImagePullBackOff | ImagePullBackOff/ErrImagePull container sayısı > 0 | 3m | critical | — |
| EKS PROD - OOMKilled | Son 5 dk restart + OOMKilled sebepli container sayısı > 0 | 1m | critical | SRE System Alerts - Critical |
| EKS PROD - PVC Pending | Pending PVC sayısı > 0 | 5m | warning | SRE System Alerts - Warning |
| EKS PROD - Failed Pods | Failed fazındaki (Evicted hariç) pod sayısı > 0 | 5m | critical | SRE System Alerts - Critical |
| EKS PROD - Waiting Containers | Config/runtime hatalı (InvalidImageName vb.) container sayısı > 0 | 5m | warning | — |
| EKS PROD - Init Container Issues | Init container'larda hata (CrashLoop/ImagePull/config) sayısı > 0 | 5m | critical | SRE System Alerts - Critical |
| EKS PROD - Pods Pending | Pending pod sayısı > 0 | 10m | warning | — |
| EKS PROD - Failed Jobs | Failed Job sayısı > 0 | 1m | warning | SRE System Alerts - Warning |
| EKS PROD - Evicted Pods | Evicted pod sayısı > 0 | 1m | warning | SRE System Alerts - Warning |
| EKS PROD - Node CPU High | Node CPU kullanımı > %80 | 10m | critical | SRE System Alerts - Critical |
| EKS PROD - Node Memory High (warning) | Node memory kullanımı > %75 (70-85 bandı) | 15m | warning | SRE System Alerts - Warning |
| EKS PROD - Pods per Node High | Node başına çalışan pod sayısı > 58 | 10m | critical | SRE System Alerts - Critical |
| EKS PROD - Deployment Replicas Unavailable | Unavailable replica'sı olan deployment sayısı ≥ 1 | 10m | critical | — |
| EKS PROD - FIRING OLMALI - Watchdog (DeadMansSwitch) | vector(1) — her zaman true | 0s | watchdog | healthchecks.io |
| EKS PROD - kube-state-metrics Down | absent(kube_node_info) — KSM veri vermiyor | 5m | critical | — |

Firing olan kural: Watchdog (DeadMansSwitch) — vector(1) ile her zaman true dönen, kasıtlı sürekli firing bir kural. Amacı healthchecks.io'ya heartbeat göndermek, Google Chat'e bildirim atmıyor. Bu normal, aksiyon gerekmiyor.

Mimari not: kube-state-metrics Down kuralı, gruptaki 17 alarmın çoğunun kaynağı olan KSM servisinin ayakta olup olmadığını izler; KSM down olduğunda diğer alarmlar no data'ya düşer.

Pod kapasite izleme üçlüsü: Node CPU High, Node Memory High ve Pods per Node High — CPU, memory ve pod/ENI limiti olmak üzere bir node'u sıkıştırabilecek 3 boyutu birlikte kapsar.

## 3. Endpoint Latency

Dashboard: AWSAPIGat, Panel 503. Toplam kural: 2 (2 normal)

| Kural | Koşul (p99 latency) | For | Severity | Receiver |
|---|---|---|---|---|
| Endpoint Latency Dağılımı (P50/P90/P95/P99) | p99 > 5000ms | 3m | critical | SRE System Alerts Critical |
| Endpoint Latency Dağılımı (P50/P90/P95/P99) (copy) | p99 > 3500ms | 3m | warning | SRE System Alerts Warning |

Dikkat: Warning kuralının açıklama metninde threshold "2000ms" yazıyor ancak gerçek threshold konfigürasyonu 3500ms. Açıklama güncellenmemiş, düzeltilmeli.

## 4. RDS

Dashboard: aws-executive-overview-v3. Toplam kural: 4

| Kural | Koşul | For | Severity |
|---|---|---|---|
| RDS Write latency Warning | Write Latency, mevcut/48s ortalama farkı normalize edilerek > 1.5 | 1m | warning |
| RDS Aurora CPU Utilization Warning | CPU Utilization > %75 | 5m | warning |
| RDS Aurora CPU Utilization Alert | CPU Utilization > %85 | 5m | critical |
| RDS Write Latency Error | Write Latency sapma oranı > 3 | 1m | critical |

Not: Write Latency kuralları statik bir threshold değil, son 5 dakikalık ortalamayı son 48 saatlik min-max aralığına göre normalize eden matematiksel bir ifade (math expression) kullanıyor — anomali tabanlı bir yaklaşım.

## Notification Receiver Özeti

| Receiver | Kural Sayısı | Kapsam |
|---|---|---|
| SRE System Alerts Critical / SRE System Alerts - Critical | 6 | Kritik seviye üretim alarmları |
| SRE System Alerts Warning / SRE System Alerts - Warning | 4 | Warning seviye alarmları |
| healthchecks.io (Watchdog) | 1 | Alerting pipeline'ın canlılığını izleme (heartbeat) |


## Bilinen Sorunlar

- Endpoint Latency warning kuralının açıklama metninde yazan threshold (2000ms), gerçek konfigürasyonla (3500ms) uyuşmuyor.
