# Grafana Alert Kuralları

Kaynak: Grafana Alerting YAML export (org 3, Alert-rules klasörü). Dört grup var: 5xx Root Cause, eks-prod-cluster, Endpoint Latency, RDS.

## 5xx Root Cause

Dashboard: 5xx-root-cause-analysis-v9. 11 kural, 2 firing, 2 no data, 9 normal.

Katmanlara ayrılmış: Authorizer, Backend Integration, genel API Gateway, .NET servis seviyesi. Her katmanda warning ve critical eşiği ayrı kurallar. ALB seviyesinde ayrıca HTTPCode_ELB_5XX_Count kuralı var. Endpoint bazlı hata sayımı için iki kural daha (warning 3.5-9.5 arası, critical >9.5).

Sadece "Top Endpoints Hata Critical" kuralı bir receiver'a bağlı (SRE System Alerts Critical), geri kalanı receiver tanımsız.

No data durumu: Authorizer/Integration kuralları CloudWatch Logs Insights sorgusuna dayanıyor. Sorgu o periyotta log bulamazsa noDataState:NoData nedeniyle "no data" görünür. Bu genelde hata olmadığı anlamına gelir, kesinti değil.

## eks-prod-cluster

Cluster: eks-prod, team: sre. 19 kural, 1 firing, 18 normal.

Kapsanan durumlar: node NotReady, deployment'ta available replica sıfır, CrashLoopBackOff, ImagePullBackOff, OOMKilled, PVC pending, failed pod, waiting container, init container hatası, pod pending, failed job, evicted pod, node CPU >%80, node memory >%75, node başına pod >58, unavailable replica.

Firing olan kural Watchdog (DeadMansSwitch) — vector(1) ile her zaman true dönen, kasıtlı sürekli firing bir kural. Amacı healthchecks.io'ya heartbeat göndermek, Google Chat'e bildirim atmıyor. Bu normal, aksiyon gerekmiyor.

kube-state-metrics Down kuralı ayrı önemli: KSM servisi düşerse gruptaki diğer kuralların çoğu veri kaynağını kaybedip no data'ya düşer.

Node CPU High, Node Memory High ve Pods per Node High üçlüsü node kapasitesini üç farklı açıdan (CPU, memory, pod/ENI limiti) izliyor.

## Endpoint Latency

Dashboard: AWSAPIGat, panel 503. 2 kural, ikisi de normal.

p99 latency: critical eşiği 5000ms, warning eşiği 3500ms.

Uyarı: warning kuralının açıklama metninde eşik "2000ms" yazıyor ama gerçek konfigürasyon 3500ms. Açıklama güncellenmemiş, düzeltilmeli.

## RDS

Dashboard: aws-executive-overview-v3. 4 kural.

Write Latency kuralları sabit eşik değil, son 5 dakikalık ortalamayı son 48 saatlik min-max aralığına göre normalize eden anomali tabanlı bir math expression kullanıyor (warning >1.5, critical sapma oranı >3).

Aurora CPU Utilization: warning >%75, critical >%85.

## Notification Receiver'lar

| Receiver | Kural sayısı | Kapsam |
|---|---|---|
| SRE System Alerts Critical (bazen tireli, bazen tiresiz yazılmış) | 6 | Kritik alarmlar |
| SRE System Alerts Warning (aynı tutarsızlık) | 4 | Warning alarmlar |
| healthchecks.io | 1 | Watchdog heartbeat |
| Tanımsız (default) | 24 | Grup/klasör seviyesi default receiver |

Receiver isimleri tutarsız — bazen boşluklu (SRE System Alerts Critical), bazen tireli (SRE System Alerts - Critical). Tek standarda geçmek (örn. sre-alerts-critical) yönlendirmeyi sadeleştirir.

## Öneriler

- 24 kuralın receiver'ı tanımsız, hepsi default'a düşüyor — kritik olanlar için en azından receiver atanmalı.
- Receiver isimlendirmesi standardize edilmeli.
- Endpoint Latency warning kuralının açıklama metni (2000ms) gerçek eşikle (3500ms) uyuşmuyor, düzeltilmeli.
