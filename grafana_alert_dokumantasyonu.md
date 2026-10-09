# Production Grafana Alert Dokümantasyonu

Sürüm: 1.0 · Tarih: 9 Ekim 2026

## 1. Kapsam ve genel görünüm

Toplam **128 kural**, **109 değerlendirmesi açık kural**, **19 duraklatılmış kural** ve **17 değerlendirme grubu** bulunur. Açık olması, sorgunun veri ürettiği veya bildirimin teslim edildiği anlamına gelmez. Duraklatılmış kurallar değerlendirilmez.

Kaynak: Pasted text(20261009-094422).txt Grafana rule exportu. Canlı durum ve bildirim kanalları için Grafana kontrolü gerekir. Bu dokümandaki eşikler condition alanı ve ifade zincirinden alınmıştır; annotation metinleriyle çelişkiler ayrıca listelenmiştir.

| Grup | Toplam | Açık | Duraklatılmış | Değerlendirme |
|---|---:|---:|---:|---|
| 5xx Root Cause | 15 | 7 | 8 | 1m |
| ALB | 2 | 2 | 0 | 1m |
| Billing | 2 | 2 | 0 | 1m |
| Customer-Heartbeat | 4 | 4 | 0 | 1m |
| DynamoDB | 10 | 10 | 0 | 1m |
| EC2 | 4 | 4 | 0 | 1m |
| ElastiCache | 10 | 10 | 0 | 1m |
| Lambda | 8 | 8 | 0 | 1m |
| MSK | 7 | 7 | 0 | 1m |
| NAT | 4 | 4 | 0 | 1m |
| Outbound | 8 | 5 | 3 | 1m |
| RDS | 11 | 9 | 2 | 1m |
| SNS | 6 | 6 | 0 | 1m |
| SQS | 7 | 7 | 0 | 1m |
| WAF | 3 | 3 | 0 | 1m |
| eks-prod-cluster | 25 | 21 | 4 | 1m |
| API-Gateway | 2 | 0 | 2 | 1m |

## 2. Alarm nasıl okunur?

- **Sorgu aralığı:** Grafananın hangi geçmiş zaman aralığını getirdiği. PromQL içindeki [5m] veya Logs Insights bin(5m) bundan ayrı bir hesaplama penceresidir.
- **Reduce:** last son örneği, mean ortalamayı, sum toplamı, max/min pencere içindeki uç değeri kullanır. Birden fazla seri varsa eşik genellikle seri bazında değerlendirilir.
- **for / pending:** Koşulun firing öncesinde sürmesi gereken süredir. for=5m, son 5 dakikanın toplamı demek değildir. Değerlendirme adımı ve sorgu penceresi nedeniyle olay başlangıcıyla bildirim zamanı aynı olmayabilir.
- **gt / lt:** Kesin olarak büyük/küçük karşılaştırmasıdır; >10, 10 değerinde tetiklenmez.
- **No Data:** OK veri yokluğunu normal kabul eder; NoData ayrı veri yokluğu durumuna geçirir; Alerting veri yokluğunu alarm durumuna geçirir.
- **Execution Error:** Error sorgu/değerlendirme hatası durumudur; Alerting hatada alarm durumuna geçirir. DatasourceError uygulamanın kesin bozulduğunu göstermez.
- **Severity:** Bildirim seviyesi labels.severity alanından okunur. Aynı eşikli warning/critical kuralları ikisi birden koşulu sağlayabilir.

## 3. Alarm geldiğinde ilk kontroller

1. Alarm UID, zaman, severity, etkilenen kaynak ve mevcut değeri kaydet. Zamanları UTC veya Türkiye saati olarak açıkça etiketle.
2. Grafana kuralında sorgunun çalıştığını ve gerçek condition değerini kontrol et. DatasourceError/No Data ise veri kaynağından başla.
3. İlgili dashboard/Explore üzerinden aynı zaman aralığını aç; namespace, pod, path, hedef veya AWS kaynak kimliğini belirle.
4. Kullanıcı etkisini hata oranı, latency, başarısız işlem ve kuyruk birikimiyle doğrula.
5. Son deploy/config değişikliği, trafik artışı ve altyapı olaylarını karşılaştır; log/trace kanıtlarını ilişkilendir.
6. Müdahale sonrası metriğin normalleşmesini, alarmın çözülmesini ve işlemlerin toparlanmasını doğrula.

## 4. Konfigürasyon farkları ve kontrol listesi

Bu maddeler yapılandırma incelemesidir; canlı sorgu çalıştırmasıyla doğrulanmalıdır.

| Öncelik | Kural / alan | Bulgular |
|---|---|---|
| Yüksek | DynamoDB grubundaki 10 kural | CloudWatch expression alanına metric math yerine JSON sorgu tanımı yazılmış; bazı model metricName/namespace alanları da boş. Eşikler tanımlı, fakat sorguların çalıştığı doğrulanmalı. JSON içindeki period/statistic değerleri model alanlarıyla yer yer farklı. |
| Yüksek | Customer-Heartbeat grubundaki 4 kural | PromQL içinde namespace=~"$namespace" bulunuyor. Dashboard değişkenine bağlı alarm kapsamı doğrulanmalı; çözümlenmeyen filtre No Data üretip noDataState=OK nedeniyle sessiz kalabilir. |
| Yüksek | Heartbeat - Error Rate Warning | Formül hata rate toplamını 100 ile çarpıyor; toplam istek rateine bölmüyor. Sonuç hata yüzdesi değil, 100 × hatalı istek/s. >5 eşiği yüzde 5 diye yorumlanmamalı. |
| Yüksek | WAF Blocked Requests Spike Warning | condition=B seçilmiş; >100 threshold C üzerinde. B bir reduce ifadesi olduğu için tanımlı eşik condition zincirinin dışında. Expression alanında ayrıca bozuk JSON metni var; kullanılabilirlik ve eşik uygulaması doğrulanmalı. |
| Yüksek | SQS DLQ Messages kuralları | Critical QueueName=*; warning boyutları boş. DLQ isim/policy filtresi görünmüyor. Genel kuyruklar DLQ olarak raporlanabilir; boş boyutlu sorguların seri döndürdüğü de doğrulanmalı. |
| Orta | RDS DB Size Spike Alert (10 Min) | A son 1800 saniyeyi sorguluyor; SEARCH periyodu 600 saniye. B=max(A), C=min(A), D=(B-C)/1073741824; koşul >15 GiB. Gerçek hesap 30 dakikalık min/max farkıdır, 10 dakikada yüzde artış değildir; küçülme de pozitif fark yaratabilir. |
| Orta | Node CPU / Node Memory / Pods per Node | Gerçek koşullar sırasıyla >80 ve for=10m; >75 ve for=15m; >50 ve for=10m. Açıklamalardaki 15m, 70–85 bandı ve 58 değeriyle uyuşmuyor. Memory warning için üst sınır yok. |
| Orta | EKS PVC Disk High (warning) | Başlık warning, severity etiketi critical. Bildirim yönlendirmesinde etiket esas alınabilir; >80 eşiği ve 5m pending var. |
| Orta | Outbound Failing Endpoints Warning/Critical | Her iki kural da >1 hedef ve for=1m. Tek hatalı hedef alarm üretmez; en az iki server_address gerekir. İki seviye aynı anda koşulu sağlar. |
| Orta | MSK Under Replicated Partitions Warning/Critical | İkisi de >0 ve for=5m; warning/critical ayrımı koşullarda yok. |
| Orta | ALB Target Response Time | A p90, D=$A>1, B=sum(D), C=B>2. Eşik doğrudan 2 saniye değildir; 5 dakikalık sorguda 1 saniyeyi aşan örnek sayısı >2 olmalı, ardından 1m pending vardır. Örnek sayısı CloudWatch periyoduna bağlıdır. |
| Orta | Billing | Warning metric period=86400, sorgu aralığı=600 saniye. Kısa sorgu aralığında günlük periyotlu verinin kullanılabilirliği kontrol edilmeli; No Data=OK maliyet izlemesini sessiz bırakabilir. |
| Orta | SNS SMS Success Rate | <95 / <90 eşikleri var; metriğin ham ölçeği ile yüzde ölçeği uyumu kontrol edilmeli. Sorguda yüzdeye dönüşüm görünmüyor. |
| Orta | No Data ve or vector(0) | Birçok alarm veri yokluğunu OK kabul ediyor. or vector(0) metrik kaybını sıfır gibi gösterebilir; ayrı scrape/veri sağlığı izlemesi gerekir. NaN/sıfıra bölme sorunlarını otomatik çözmez. |
| Orta | Watchdog | condition=A; C threshold tanımlı fakat seçili değil. A=vector(1) sürekli pozitif sinyal amaçlıyor. Alarm değerlendirmesi ve heartbeat teslimi uçtan uca doğrulanmalı; contact point/notification policy exportta bulunmuyor. |
| Düşük | 5XX RCA açıklamaları | Authorizer ve Integration critical gerçek eşik >4, .NET critical >9. API GW Warning >0, Critical >9. Bazı annotation eşikleri farklı; gerçek condition esas alınmalı. Success Rate yalnızca 5xxi hata saydığı için 4xxler başarı tarafında kalıyor. |
| Düşük | EKS açıklama temizliği | Pods per Node açıklamasında sohbet/talimat metni bulunuyor. Certificate Expiring gerçek koşul <15 gün ve alt sınır yok; Certificate Not Ready for=10m, CoreDNS for=5m. |
| Düşük | Kaynak kapsamı ve adlar | cluster=eks-prod etiketi tek başına PromQL filtresi değildir. Prod kapsamı datasource ve query üzerinden doğrulanmalı. RDS grubundaki DynamoDB kuralları, Write Latency aliasları ve severity/title eşleşmeleri düzenlenmeli; datasourceUid ile model.datasource.uid farklı olan kurallar kontrol edilmeli. |

## 5. Grup bazında alarm kataloğu

### 5xx Root Cause

API Gateway ve uygulama hatalarını katmana göre ayırır; DLQ kuralı işlenemeyen mesaj loglarını izler.

**Alarm geldiğinde:** İlgili path, method, tenant ve correlationId ile access loglarını incele. Authorizer hatasında Lambda loglarına; integration hatasında ALB target health ve backend podlarına; .NET hatasında Tempo trace ve uygulama exception kaydına bak. DLQ için topic, partition, offset ve iş kaydını eşleştir; yeniden işleme öncesinde sebebi ve idempotency davranışını doğrula.

| Kural | Severity | Durum | Gerçek condition | Pending |
|---|---|---|---|---|
| 5XX RCA - Authorizer Failures - Warning | warning | Duraklatılmış | B > 0 | 5m |
| 5XX RCA - Authorizer Failures - Critical | critical | Duraklatılmış | B > 4 | 2m |
| 5XX RCA - Integration Failures - Warning | warning | Açık | B > 0 | 5m |
| 5XX RCA - Integration Failures - Critical | critical | Açık | B > 4 | 2m |
| 5XX RCA - API GW 5XX - Warning | warning | Duraklatılmış | B > 0 | 5m |
| 5XX RCA - API GW 5XX - Critical | critical | Duraklatılmış | B > 9 | 2m |
| 5XX RCA - .NET OTel 5XX - Warning | warning | Açık | B > 0 | 5m |
| 5XX RCA - .NET OTel 5XX - Critical | critical | Açık | B > 9 | 2m |
| 📊Top Endpoints Hata Warning | warning | Açık | B aralığında 3.5, 9.5 | 5m |
| 📊Top Endpoints Hata Critical Alert | critical | Açık | B > 9.5, 9.5 | 5m |
| 5XX RCA - Container Errors - Warning | warning | Duraklatılmış | B > 0 | 5m |
| 5XX RCA - Container Errors - Critical | critical | Duraklatılmış | B > 9 | 2m |
| 5XX RCA - Success Rate - Warning | warning | Duraklatılmış | B < 95 | 5m |
| 5XX RCA - Success Rate - Critical | critical | Duraklatılmış | B < 90 | 3m |
| DLQ > 0 Critical | critical | Açık | B > 0 | 1m |

#### Sorgular, kapsam ve değerlendirme zincirleri

##### 5XX RCA - Authorizer Failures - Warning

UID: `cfuyg7uzpaf40e` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

```text
fields errorResponseType, path
| filter status = '500' and errorResponseType like /AUTHORIZER/
| stats count(*) as cnt by path, errorResponseType, bin(5m)
```

Log grupları:

- `/aws/apigateway/era-plt-api-gateway-prod/access-logs`
- `/aws/apigateway/era-api-gateway-prod/access-logs`

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.
Dashboard: https://grafana-new.eradev.cloud/d/5xx-root-cause-analysis-v9


##### 5XX RCA - Authorizer Failures - Critical

UID: `cfuygklh5uoe8f` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

```text
fields errorResponseType, path
| filter status = '500' and errorResponseType like /AUTHORIZER/
| stats count(*) as cnt by path, errorResponseType, bin(5m)
```

Log grupları:

- `/aws/apigateway/era-plt-api-gateway-prod/access-logs`
- `/aws/apigateway/era-api-gateway-prod/access-logs`

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.
Dashboard: https://grafana-new.eradev.cloud/d/5xx-root-cause-analysis-v9


##### 5XX RCA - Integration Failures - Warning

UID: `efv4t0l67fw8wf` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

```text
fields errorResponseType, path
| filter status = '500' and errorResponseType not like /AUTHORIZER/ and integrationStatus not like /200/
| stats count(*) as cnt by path, errorResponseType, bin(5m)
```

Log grupları:

- `/aws/apigateway/era-plt-api-gateway-prod/access-logs`
- `/aws/apigateway/era-api-gateway-prod/access-logs`

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.
Dashboard: https://grafana-new.eradev.cloud/d/5xx-root-cause-analysis-v9


##### 5XX RCA - Integration Failures - Critical

UID: `bfv4t3swokrggd` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

```text
fields errorResponseType, path
| filter status = '500' and errorResponseType not like /AUTHORIZER/ and integrationStatus not like /200/
| stats count(*) as cnt by path, errorResponseType, bin(5m)
```

Log grupları:

- `/aws/apigateway/era-plt-api-gateway-prod/access-logs`
- `/aws/apigateway/era-api-gateway-prod/access-logs`

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.
Dashboard: https://grafana-new.eradev.cloud/d/5xx-root-cause-analysis-v9


##### 5XX RCA - API GW 5XX - Warning

UID: `dfv4trnq2ax34a` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/ApiGateway/5XXError` · statistic=`Sum` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{}`.

```text
SEARCH('{AWS/ApiGateway,ApiName,Stage,Method,Resource} MetricName="5XXError"', 'Sum', 60)
```

Log grupları:

- `/aws/apigateway/era-plt-api-gateway-prod/access-logs`
- `/aws/apigateway/era-api-gateway-prod/access-logs`

- **B:** type=reduce, expression=`A`, reducer=sum.
- **C:** type=threshold, expression=`B`, reducer=—.
Dashboard: https://grafana-new.eradev.cloud/d/5xx-root-cause-analysis-v9


##### 5XX RCA - API GW 5XX - Critical

UID: `efv4tv267m4n4e` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/ApiGateway/5XXError` · statistic=`Sum` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{}`.

```text
SEARCH('{AWS/ApiGateway,ApiName,Stage,Method,Resource} MetricName="5XXError"', 'Sum', 60)
```

Log grupları:

- `/aws/apigateway/era-plt-api-gateway-prod/access-logs`
- `/aws/apigateway/era-api-gateway-prod/access-logs`

- **B:** type=reduce, expression=`A`, reducer=sum.
- **C:** type=threshold, expression=`B`, reducer=—.
Dashboard: https://grafana-new.eradev.cloud/d/5xx-root-cause-analysis-v9


##### 5XX RCA - .NET OTel 5XX - Warning

UID: `bfv59xl5lci68c` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
sum by (service_name, pod, http_route) (
  increase(http_server_request_duration_seconds_count{
    namespace=~".*",
    http_response_status_code=~"5..",
    http_route!~"/health.*|/hubs.*"
  }[5m])
)
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.
Dashboard: https://grafana-new.eradev.cloud/d/5xx-root-cause-analysis-v9


##### 5XX RCA - .NET OTel 5XX - Critical

UID: `cfv5abd1l69s0c` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
sum by (service_name, pod, http_route) (
  increase(http_server_request_duration_seconds_count{
    namespace=~".*",
    http_response_status_code=~"5..",
    http_route!~"/health.*|/hubs.*"
  }[5m])
)
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.
Dashboard: https://grafana-new.eradev.cloud/d/5xx-root-cause-analysis-v9


##### 📊Top Endpoints Hata Warning

UID: `afuxtenrb01s0e` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-1800s → now-0s.

```text
fields httpMethod, path, status
| filter httpMethod != "OPTIONS"
| stats sum(status >= 500) as error_count by httpMethod, path, bin(1m)
```

Log grupları:

- `/aws/apigateway/era-api-gateway-prod/access-logs`
- `/aws/apigateway/era-plt-api-gateway-prod/access-logs`

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.
Dashboard: https://grafana-new.eradev.cloud/d/AWSAPIGat/aws-api-gateway?orgId=3&refresh=1m


##### 📊Top Endpoints Hata Critical Alert

UID: `ffuxvkpb32dj4d` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-1800s → now-0s.

```text
fields httpMethod, path, status
| filter httpMethod != "OPTIONS"
| stats sum(status >= 500) as error_count by httpMethod, path, bin(1m)
```

Log grupları:

- `/aws/apigateway/era-api-gateway-prod/access-logs`
- `/aws/apigateway/era-plt-api-gateway-prod/access-logs`

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### 5XX RCA - Container Errors - Warning

UID: `efvg2fe6691xca` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

```text
filter @message like /(?i)(unhandled exception|fail:.*ExceptionHandler)/
| parse @logGroup /\/(?<service_name>[^\/]+)$/
| stats count(*) as error_count by service_name, bin(5m)
```

Log grupları:

- `/aws/containerinsights/era-eks-prod/admin-panel-backend`
- `/aws/containerinsights/era-eks-prod/app-reports`
- `/aws/containerinsights/era-eks-prod/e-commerce`
- `/aws/containerinsights/era-eks-prod/online-store-backend`
- `/aws/containerinsights/era-eks-prod/plt-comm-account`
- `/aws/containerinsights/era-eks-prod/plt-comm-auth`
- `/aws/containerinsights/era-eks-prod/plt-comm-catalog`
- `/aws/containerinsights/era-eks-prod/plt-comm-common`
- `/aws/containerinsights/era-eks-prod/plt-comm-customer`
- `/aws/containerinsights/era-eks-prod/plt-comm-datasynclayer`
- `/aws/containerinsights/era-eks-prod/plt-comm-datasynclayer-webhook`
- `/aws/containerinsights/era-eks-prod/plt-comm-datasynclayer-websocket`
- `/aws/containerinsights/era-eks-prod/plt-comm-e-invoice`
- `/aws/containerinsights/era-eks-prod/plt-comm-edoc-validator`
- `/aws/containerinsights/era-eks-prod/plt-comm-import-export`
- `/aws/containerinsights/era-eks-prod/plt-comm-inventory`
- `/aws/containerinsights/era-eks-prod/plt-comm-invoice`
- `/aws/containerinsights/era-eks-prod/plt-comm-lens`
- `/aws/containerinsights/era-eks-prod/plt-comm-notification`
- `/aws/containerinsights/era-eks-prod/plt-comm-openapi`
- `/aws/containerinsights/era-eks-prod/plt-comm-order`
- `/aws/containerinsights/era-eks-prod/plt-comm-payment`
- `/aws/containerinsights/era-eks-prod/plt-comm-shipping`
- `/aws/containerinsights/era-eks-prod/plt-comm-shopping-cart`
- `/aws/containerinsights/era-eks-prod/plt-comm-tax-free`
- `/aws/containerinsights/era-eks-prod/plt-comm-workflow-manager`
- `/aws/containerinsights/era-eks-prod/pos-backend`

- **B:** type=reduce, expression=`A`, reducer=max.
- **C:** type=threshold, expression=`B`, reducer=—.
Dashboard: https://grafana-new.eradev.cloud/d/5xx-root-cause-analysis-v9


##### 5XX RCA - Container Errors - Critical

UID: `bfvg2j0yycbnka` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

```text
filter @message like /(?i)(unhandled exception|fail:.*ExceptionHandler)/
| parse @logGroup /\/(?<service_name>[^\/]+)$/
| stats count(*) as error_count by service_name, bin(5m)
```

Log grupları:

- `/aws/apigateway/era-api-gateway-prod/access-logs`
- `/aws/apigateway/era-plt-api-gateway-prod/access-logs`
- `/aws/apigateway/welcome`
- `/aws/containerinsights/era-eks-prod/admin-panel-backend`
- `/aws/containerinsights/era-eks-prod/app-reports`
- `/aws/containerinsights/era-eks-prod/e-commerce`
- `/aws/containerinsights/era-eks-prod/online-store-backend`
- `/aws/containerinsights/era-eks-prod/plt-comm-account`
- `/aws/containerinsights/era-eks-prod/plt-comm-auth`
- `/aws/containerinsights/era-eks-prod/plt-comm-catalog`
- `/aws/containerinsights/era-eks-prod/plt-comm-common`
- `/aws/containerinsights/era-eks-prod/plt-comm-customer`
- `/aws/containerinsights/era-eks-prod/plt-comm-datasynclayer`
- `/aws/containerinsights/era-eks-prod/plt-comm-datasynclayer-webhook`
- `/aws/containerinsights/era-eks-prod/plt-comm-datasynclayer-websocket`
- `/aws/containerinsights/era-eks-prod/plt-comm-e-invoice`
- `/aws/containerinsights/era-eks-prod/plt-comm-edoc-validator`
- `/aws/containerinsights/era-eks-prod/plt-comm-import-export`
- `/aws/containerinsights/era-eks-prod/plt-comm-inventory`
- `/aws/containerinsights/era-eks-prod/plt-comm-invoice`
- `/aws/containerinsights/era-eks-prod/plt-comm-lens`
- `/aws/containerinsights/era-eks-prod/plt-comm-notification`
- `/aws/containerinsights/era-eks-prod/plt-comm-openapi`
- `/aws/containerinsights/era-eks-prod/plt-comm-order`
- `/aws/containerinsights/era-eks-prod/plt-comm-payment`
- `/aws/containerinsights/era-eks-prod/plt-comm-shipping`
- `/aws/containerinsights/era-eks-prod/plt-comm-shopping-cart`
- `/aws/containerinsights/era-eks-prod/plt-comm-tax-free`
- `/aws/containerinsights/era-eks-prod/plt-comm-workflow-manager`
- `/aws/containerinsights/era-eks-prod/pos-backend`
- `/aws/eks/era-eks-prod/cluster`

- **B:** type=reduce, expression=`A`, reducer=max.
- **C:** type=threshold, expression=`B`, reducer=—.
Dashboard: https://grafana-new.eradev.cloud/d/5xx-root-cause-analysis-v9


##### 5XX RCA - Success Rate - Warning

UID: `efvg3blsfsg74a` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

```text
fields status
| stats count(*) as total, sum(status >= 500) as errors by bin(5m)
| fields round((total - errors) * 100.0 / total, 2) as success_rate
```

Log grupları:

- `/aws/apigateway/era-api-gateway-prod/access-logs`
- `/aws/apigateway/era-plt-api-gateway-prod/access-logs`

- **B:** type=reduce, expression=`A`, reducer=min.
- **C:** type=threshold, expression=`B`, reducer=—.
Dashboard: https://grafana-new.eradev.cloud/d/5xx-root-cause-analysis-v9


##### 5XX RCA - Success Rate - Critical

UID: `bfvg3eghl04jkc` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

```text
fields status
| stats count(*) as total, sum(status >= 500) as errors by bin(5m)
| fields round((total - errors) * 100.0 / total, 2) as success_rate
```

Log grupları:

- `/aws/apigateway/era-api-gateway-prod/access-logs`
- `/aws/apigateway/era-plt-api-gateway-prod/access-logs`

- **B:** type=reduce, expression=`A`, reducer=min.
- **C:** type=threshold, expression=`B`, reducer=—.
Dashboard: https://grafana-new.eradev.cloud/d/5xx-root-cause-analysis-v9


##### DLQ > 0 Critical

UID: `ffvjyqyh6v75sb` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-300s → now-0s.

```text
filter EventId.Id = 5006
| parse @log "*:*" as hesapId, temiz_log_grubu
| parse temiz_log_grubu "*/" as cop_kisim, Servis_Adi
| stats count(*) as Adet by Servis_Adi
| sort Adet desc

```

Log grupları:

- `/aws/containerinsights/era-eks-prod/admin-panel-backend`
- `/aws/containerinsights/era-eks-prod/data-process`
- `/aws/containerinsights/era-eks-prod/e-commerce`
- `/aws/containerinsights/era-eks-prod/online-store-backend`
- `/aws/containerinsights/era-eks-prod/plt-comm-account`
- `/aws/containerinsights/era-eks-prod/plt-comm-auth`
- `/aws/containerinsights/era-eks-prod/plt-comm-catalog`
- `/aws/containerinsights/era-eks-prod/plt-comm-common`
- `/aws/containerinsights/era-eks-prod/plt-comm-customer`
- `/aws/containerinsights/era-eks-prod/plt-comm-datasynclayer`
- `/aws/containerinsights/era-eks-prod/plt-comm-datasynclayer-webhook`
- `/aws/containerinsights/era-eks-prod/plt-comm-datasynclayer-websocket`
- `/aws/containerinsights/era-eks-prod/plt-comm-e-invoice`
- `/aws/containerinsights/era-eks-prod/plt-comm-edoc-validator`
- `/aws/containerinsights/era-eks-prod/plt-comm-import-export`
- `/aws/containerinsights/era-eks-prod/plt-comm-inventory`
- `/aws/containerinsights/era-eks-prod/plt-comm-invoice`
- `/aws/containerinsights/era-eks-prod/plt-comm-lens`
- `/aws/containerinsights/era-eks-prod/plt-comm-notification`
- `/aws/containerinsights/era-eks-prod/plt-comm-openapi`
- `/aws/containerinsights/era-eks-prod/plt-comm-order`
- `/aws/containerinsights/era-eks-prod/plt-comm-payment`
- `/aws/containerinsights/era-eks-prod/plt-comm-shipping`
- `/aws/containerinsights/era-eks-prod/plt-comm-shopping-cart`
- `/aws/containerinsights/era-eks-prod/plt-comm-tax-free`
- `/aws/containerinsights/era-eks-prod/plt-comm-workflow-manager`
- `/aws/eks/era-eks-prod/cluster`

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

### ALB

Load balancer kaynaklı 5xx ve target yanıt süresini izler.

**Alarm geldiğinde:** ALB, listener, target group ve target health durumunu kontrol et. ELB kaynaklı hata ile target kaynaklı hatayı ayır; access logları, backend latency ve pod readiness durumunu karşılaştır.

| Kural | Severity | Durum | Gerçek condition | Pending |
|---|---|---|---|---|
| ALB Target Response Time | critical | Açık | B > 2 | 1m |
| ALB Error Rates 5xx | critical | Açık | B > 0, 0 | 1m |

#### Sorgular, kapsam ve değerlendirme zincirleri

##### ALB Target Response Time

UID: `cfvnc6nxmyg3kf` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-300s → now-0s.

Metrik model alanları: `AWS/ApplicationELB/TargetResponseTime` · statistic=`p90` · period=`60` · region=`eu-central-1`.

Boyutlar: `{"LoadBalancer": ["*"]}`.

- **B:** type=reduce, expression=`D`, reducer=sum.
- **C:** type=threshold, expression=`B`, reducer=—.
- **D:** type=math, expression=`$A > 1`, reducer=—.

##### ALB Error Rates 5xx

UID: `bfv21ecvi7wu8c` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-300s → now-0s.

Metrik model alanları: `AWS/ApplicationELB/HTTPCode_ELB_5XX_Count` · statistic=`Sum` · period=`60` · region=`eu-central-1`.

Boyutlar: `{"LoadBalancer": ["*"]}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

### Billing

USD cinsinden EstimatedCharges metriğini sabit eşiklerle izler.

**Alarm geldiğinde:** Billing ekranında maliyetin dönemini ve kapsamını doğrula. Cost Explorer üzerinde servis ve hesap bazında artışı incele. 115/130 eşiklerinin bütçe ile uyumunu kontrol et.

| Kural | Severity | Durum | Gerçek condition | Pending |
|---|---|---|---|---|
| Billing Estimated Charges Warning | warning | Açık | B > 115 | 5m |
| Billing Estimated Charges Critical | critical | Açık | B > 130 | 5m |

#### Sorgular, kapsam ve değerlendirme zincirleri

##### Billing Estimated Charges Warning

UID: `efxtiwymeemm8f` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/Billing/EstimatedCharges` · statistic=`Average` · period=`86400` · region=`us-east-1`.

Boyutlar: `{"Currency": "USD"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### Billing Estimated Charges Critical

UID: `bfxtkdtud9szka` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/Billing/EstimatedCharges` · statistic=`Average` · period=`otomatik/boş` · region=`us-east-1`.

Boyutlar: `{"Currency": "USD"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

### Customer-Heartbeat

Container bellek limiti kullanımı, HTTP client hata hızı ve pod restartlarını izler.

**Alarm geldiğinde:** Önce namespace filtresinin çözüldüğünü ve sorgunun veri döndürdüğünü doğrula. İlgili podun memory limit, working set ve son termination kaydını incele; HTTP client hatalarını hedef servis ve status code ile ayır.

| Kural | Severity | Durum | Gerçek condition | Pending |
|---|---|---|---|---|
| Heartbeat - Memory Utilization Warning | warning | Açık | B > 80 | 1m |
| Heartbeat - Memory Utilization Critical | critical | Açık | B > 90 | 1m |
| Heartbeat - Error Rate Warning | warning | Açık | B > 5 | 1m |
| Heartbeat - Pod Restart Warning | warning | Açık | B > 0 | 1m |

#### Sorgular, kapsam ve değerlendirme zincirleri

##### Heartbeat - Memory Utilization Warning

UID: `ffxtmz0bxdvy8b` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
(container_memory_working_set_bytes{namespace=~"$namespace", container!="POD", container!=""} / (container_spec_memory_limit_bytes{namespace=~"$namespace", container!="POD", container!=""} > 0)) * 100
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### Heartbeat - Memory Utilization Critical

UID: `afxtnpptjv1fka` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
(container_memory_working_set_bytes{namespace=~"$namespace", container!="POD", container!=""} / (container_spec_memory_limit_bytes{namespace=~"$namespace", container!="POD", container!=""} > 0)) * 100
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### Heartbeat - Error Rate Warning

UID: `ffxtopglfbnr4e` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
100 * sum by(service_name)(rate(http_client_request_duration_seconds_count{namespace=~"$namespace", http_response_status_code=~"4..|5.."}[5m]))
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### Heartbeat - Pod Restart Warning

UID: `efxtpdcmn9nuof` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
clamp_min(sum by(namespace,pod)(increase(kube_pod_container_status_restarts_total{namespace=~"$namespace"}[15m])), 0)
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

### DynamoDB

SystemErrors, throttling, UserErrors ve shopping_carts GetItem/UpdateItem gecikmelerini hedefler.

**Alarm geldiğinde:** Önce sorgu biçimini ve TableName/Operation boyutlarını doğrula. Kapasite, throttling, yoğun partition ve uygulama retry davranışını incele. UserErrors için uygulama hata kodunu bul; latency artışını aynı zaman aralığındaki trafikle karşılaştır.

| Kural | Severity | Durum | Gerçek condition | Pending |
|---|---|---|---|---|
| DynamoDB System Errors Warning | warning | Açık | B > 0 | 5m |
| DynamoDB System Errors Critical | critical | Açık | B > 10 | 5m |
| DynamoDB Throttled Requests Warning | warning | Açık | B > 1 | 5m |
| DynamoDB Throttled Requests Critical | critical | Açık | B > 10 | 5m |
| DynamoDB User Errors Warning | warning | Açık | B > 50 | 5m |
| DynamoDB User Errors Critical | critical | Açık | B > 200 | 5m |
| DynamoDB Read Latency Warning | warning | Açık | B > 50 | 5m |
| DynamoDB Read Latency Critical | critical | Açık | B > 200 | 5m |
| DynamoDB Write Latency Critical | critical | Açık | B > 200 | 5m |
| DynamoDB Write Latency Warning | warning | Açık | B > 50 | 5m |

#### Sorgular, kapsam ve değerlendirme zincirleri

##### DynamoDB System Errors Warning

UID: `dfxr5cff6m77kf` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/DynamoDB/SystemErrors` · statistic=`Average` · period=`60` · region=`eu-central-1`.

Boyutlar: `{}`.

```text
{"namespace":"AWS/DynamoDB","metricName":"SystemErrors","region":"eu-central-1","statistic":"Average","period":"300","refId":"A"}
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### DynamoDB System Errors Critical

UID: `dfxr667u09se8a` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

```text
{"namespace":"AWS/DynamoDB","metricName":"SystemErrors","region":"eu-central-1","statistic":"Average","period":"300","refId":"A"}
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### DynamoDB Throttled Requests Warning

UID: `efxr6zi52f5kwd` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

```text
{"namespace":"AWS/DynamoDB","metricName":"ThrottledRequests","region":"eu-central-1","statistic":"Sum","period":"300","refId":"A"}
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### DynamoDB Throttled Requests Critical

UID: `ffxrestgyascgd` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

```text
{"namespace":"AWS/DynamoDB","metricName":"ThrottledRequests","region":"eu-central-1","statistic":"Sum","period":"300","refId":"A"}
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### DynamoDB User Errors Warning

UID: `ffxrfn2gy6hhcd` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

```text
{"namespace":"AWS/DynamoDB","metricName":"UserErrors","region":"eu-central-1","statistic":"Sum","period":300,"refId":"A"}
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### DynamoDB User Errors Critical

UID: `cfxrgli1wp9fkd` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

```text
{"namespace":"AWS/DynamoDB","metricName":"UserErrors","region":"eu-central-1","statistic":"Sum","period":300,"refId":"A"}
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### DynamoDB Read Latency Warning

UID: `bfxrjexohhjwgc` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

```text
{"namespace":"AWS/DynamoDB","metricName":"SuccessfulRequestLatency","dimensions":{"TableName":"shopping_carts","Operation":"GetItem"},"region":"eu-central-1","statistic":"Average","period":300,"refId":"A"}
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### DynamoDB Read Latency Critical

UID: `afxrjl4nt6j9ce` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

```text
{"namespace":"AWS/DynamoDB","metricName":"SuccessfulRequestLatency","dimensions":{"TableName":"shopping_carts","Operation":"GetItem"},"region":"eu-central-1","statistic":"Average","period":300,"refId":"A"}
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### DynamoDB Write Latency Critical

UID: `ffxrkg35g9ddsb` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

```text
{"namespace":"AWS/DynamoDB","metricName":"SuccessfulRequestLatency","dimensions":{"TableName":"shopping_carts","Operation":"UpdateItem"},"region":"eu-central-1","statistic":"Average","period":300,"refId":"A"}
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### DynamoDB Write Latency Warning

UID: `afxrkpctsgrnkc` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

```text
{"namespace":"AWS/DynamoDB","metricName":"SuccessfulRequestLatency","dimensions":{"TableName":"shopping_carts","Operation":"UpdateItem"},"region":"eu-central-1","statistic":"Average","period":300,"refId":"A"}
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

### EC2

Instance status check ve CPU kullanımını izler.

**Alarm geldiğinde:** Instance kimliğini doğrula; system ve instance status check sonuçlarını ayır. CPU grafiğini süreç yükü, deploy ve trafikle karşılaştır; EKS node ise kubectl describe node ve workload dağılımını incele.

| Kural | Severity | Durum | Gerçek condition | Pending |
|---|---|---|---|---|
| EC2 Status Check Failed Warning | warning | Açık | B > 0 | 5m |
| EC2 Status Check Failed Critical | critical | Açık | B > 0 | 15m |
| EC2 CPU Utilization Warning | warning | Açık | B > 75 | 5m |
| EC2 CPU Utilization Critical | critical | Açık | B > 90 | 5m |

#### Sorgular, kapsam ve değerlendirme zincirleri

##### EC2 Status Check Failed Warning

UID: `dfxtsdnebw45cc` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/EC2/StatusCheckFailed` · statistic=`Average` · period=`1m` · region=`eu-central-1`.

Boyutlar: `{}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### EC2 Status Check Failed Critical

UID: `cfxttebju0ao0d` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/EC2/StatusCheckFailed` · statistic=`Maximum` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### EC2 CPU Utilization Warning

UID: `cfxtujj6wvncwd` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/EC2/CPUUtilization` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### EC2 CPU Utilization Critical

UID: `bfxtvlzfj7n5sa` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/EC2/CPUUtilization` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

### ElastiCache

Redis CPU, engine CPU, memory, eviction, replication lag, swap ve bağlantı sayılarını izler.

**Alarm geldiğinde:** Cluster ve node kimliğini bul. Engine CPU ile host CPUyu ayrı değerlendir. Memory artışında key sayısı, TTL ve eviction trendine; lag için replica bağlantısına; connections için connection pool ve bağlantı sızıntısına bak.

| Kural | Severity | Durum | Gerçek condition | Pending |
|---|---|---|---|---|
| ElastiCache Redis CPU Utilization Warning | warning | Açık | B > 4 | 5m |
| ElastiCache Redis CPU Utilization Critical | critical | Açık | B > 7 | 5m |
| ElastiCache Redis Engine CPU Warning | warning | Açık | B > 1 | 5m |
| ElastiCache Redis Engine CPU Critical | critical | Açık | B > 2 | 5m |
| ElastiCache Redis Memory Warning | warning | Açık | B > 80 | 5m |
| ElastiCache Redis Memory Critical | critical | Açık | B > 90 | 5m |
| ElastiCache Redis Evictions Warning | warning | Açık | B > 0 | 5m |
| ElastiCache Redis Replication Lag Warning | warning | Açık | B > 5 | 5m |
| ElastiCache Redis Swap Usage Warning | warning | Açık | B > 0 | 5m |
| ElastiCache Redis Connections High Warning | warning | Açık | B > 250 | 5m |

#### Sorgular, kapsam ve değerlendirme zincirleri

##### ElastiCache Redis CPU Utilization Warning

UID: `afxqbsm3gm58gb` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/ElastiCache/CPUUtilization` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"CacheClusterId": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### ElastiCache Redis CPU Utilization Critical

UID: `cfxqc6t4y9n9ca` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/ElastiCache/CPUUtilization` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"CacheClusterId": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### ElastiCache Redis Engine CPU Warning

UID: `ffxqcmcw34we8a` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/ElastiCache/EngineCPUUtilization` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"CacheClusterId": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### ElastiCache Redis Engine CPU Critical

UID: `ffxqcvxbmlo8we` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/ElastiCache/EngineCPUUtilization` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"CacheClusterId": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### ElastiCache Redis Memory Warning

UID: `afxqd5urj124ge` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/ElastiCache/DatabaseMemoryUsagePercentage` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"CacheClusterId": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### ElastiCache Redis Memory Critical

UID: `efxqdhn9fttz4c` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/ElastiCache/DatabaseMemoryUsagePercentage` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"CacheClusterId": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### ElastiCache Redis Evictions Warning

UID: `efxqdtnp00lc0d` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/ElastiCache/Evictions` · statistic=`Sum` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"CacheClusterId": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### ElastiCache Redis Replication Lag Warning

UID: `ffxqe2y1tq5moc` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/ElastiCache/ReplicationLag` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"CacheClusterId": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### ElastiCache Redis Swap Usage Warning

UID: `afxqei8449wcgc` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/ElastiCache/SwapUsage` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"CacheClusterId": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### ElastiCache Redis Connections High Warning

UID: `cfxqewkt25o8wb` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/ElastiCache/CurrConnections` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"CacheClusterId": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

### Lambda

Function bazında hata, throttle, duration ve stream iterator age izler.

**Alarm geldiğinde:** FunctionName ve log stream üzerinden exception/timeout kaydını bul. Throttle için concurrency ve quota durumunu; Duration için dış bağımlılıkları; IteratorAge için event source mapping ve batch işleme hızını incele.

| Kural | Severity | Durum | Gerçek condition | Pending |
|---|---|---|---|---|
| Lambda Errors Warning | warning | Açık | B > 0 | 5m |
| Lambda Errors Critical | critical | Açık | B > 10 | 5m |
| Lambda Throttles Warning | warning | Açık | B > 0 | 5m |
| Lambda Throttles Critical | critical | Açık | B > 10 | 5m |
| Lambda Duration High Warning | warning | Açık | B > 3000 | 5m |
| Lambda Duration High Critical | critical | Açık | B > 10000 | 5m |
| Lambda Iterator Age Warning | warning | Açık | B > 60000 | 5m |
| Lambda Iterator Age Critical | critical | Açık | B > 300000 | 5m |

#### Sorgular, kapsam ve değerlendirme zincirleri

##### Lambda Errors Warning

UID: `cfxr1y1exrv9cb` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/Lambda/Errors` · statistic=`Sum` · period=`60` · region=`eu-central-1`.

Boyutlar: `{"FunctionName": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### Lambda Errors Critical

UID: `cfxr29coasqo0e` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/Lambda/Errors` · statistic=`Sum` · period=`60` · region=`eu-central-1`.

Boyutlar: `{"FunctionName": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### Lambda Throttles Warning

UID: `efxr2hagc3g1sf` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/Lambda/Throttles` · statistic=`Sum` · period=`60` · region=`eu-central-1`.

Boyutlar: `{"FunctionName": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### Lambda Throttles Critical

UID: `dfxr2quyzxngge` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/Lambda/Throttles` · statistic=`Sum` · period=`60` · region=`eu-central-1`.

Boyutlar: `{"FunctionName": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### Lambda Duration High Warning

UID: `cfxr3dfhzwcu8d` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/Lambda/Duration` · statistic=`Average` · period=`60` · region=`eu-central-1`.

Boyutlar: `{"FunctionName": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### Lambda Duration High Critical

UID: `ffxr3ntfqoq2ob` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/Lambda/Duration` · statistic=`Average` · period=`60` · region=`eu-central-1`.

Boyutlar: `{"FunctionName": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### Lambda Iterator Age Warning

UID: `efxr3su6roh6od` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/Lambda/IteratorAge` · statistic=`Average` · period=`60` · region=`eu-central-1`.

Boyutlar: `{"FunctionName": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### Lambda Iterator Age Critical

UID: `ffxr3z16uuccge` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/Lambda/IteratorAge` · statistic=`Average` · period=`60` · region=`eu-central-1`.

Boyutlar: `{"FunctionName": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

### MSK

Kafka partition replikasyonu, offline partition, disk ve CpuUser metriğini izler.

**Alarm geldiğinde:** Cluster ve broker kimliğini doğrula. Broker sağlık durumunu, partition ISR ve disk trendini incele. Producer/consumer loglarını ve bağlantı hatalarını karşılaştır; disk sorunu için retention ve trafik artışını değerlendir.

| Kural | Severity | Durum | Gerçek condition | Pending |
|---|---|---|---|---|
| MSK Under Replicated Partitions Warning | warning | Açık | B > 0 | 5m |
| MSK Under Replicated Partitions Critical | critical | Açık | B > 0 | 5m |
| MSK Offline Partitions Critical | critical | Açık | B > 0 | 1m |
| MSK Kafka Data Logs Disk Used Warning | warning | Açık | B > 80 | 5m |
| MSK Kafka Data Logs Disk Used Critical | critical | Açık | B > 90 | 5m |
| MSK Broker CPU Warning | warning | Açık | B > 75 | 5m |
| MSK Broker CPU Critical | critical | Açık | B > 90 | 5m |

#### Sorgular, kapsam ve değerlendirme zincirleri

##### MSK Under Replicated Partitions Warning

UID: `afxqq91d4tgcgc` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/Kafka/UnderReplicatedPartitions` · statistic=`Maximum` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"Broker ID": "*", "Cluster Name": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### MSK Under Replicated Partitions Critical

UID: `cfxqru0v8owsgc` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/Kafka/UnderReplicatedPartitions` · statistic=`Maximum` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"Broker ID": "*", "Cluster Name": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### MSK Offline Partitions Critical

UID: `dfxqthrupzgn4a` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/Kafka/OfflinePartitionsCount` · statistic=`Maximum` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"Cluster Name": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### MSK Kafka Data Logs Disk Used Warning

UID: `ffxqvcsj8bh8gf` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/Kafka/KafkaDataLogsDiskUsed` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"Cluster Name": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### MSK Kafka Data Logs Disk Used Critical

UID: `cfxqwzw6orawwb` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/Kafka/KafkaDataLogsDiskUsed` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"Cluster Name": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### MSK Broker CPU Warning

UID: `efxqyw0e1962od` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/Kafka/CpuUser` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"Cluster Name": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### MSK Broker CPU Critical

UID: `bfxr08x9fxipsb` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/Kafka/CpuUser` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"Cluster Name": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

### NAT

NAT Gateway port allocation hataları ve düşen paketleri izler.

**Alarm geldiğinde:** NatGatewayId ile etkilenen subnet/route kapsamını bul. Bağlantı sayısı ve trafik artışını incele; uygulamada dış servis timeoutlarını karşılaştır. Paket düşüşünü tek başına tam kesinti olarak yorumlama.

| Kural | Severity | Durum | Gerçek condition | Pending |
|---|---|---|---|---|
| NAT Error Port Allocation Warning | warning | Açık | B > 0 | 5m |
| NAT Error Port Allocation Critical | critical | Açık | B > 10 | 5m |
| NAT Packets Dropped Warning | warning | Açık | B > 0 | 5m |
| NAT Packets Dropped Critical | critical | Açık | B > 10 | 5m |

#### Sorgular, kapsam ve değerlendirme zincirleri

##### NAT Error Port Allocation Warning

UID: `ffxr9n3a882rkd` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/NATGateway/ErrorPortAllocation` · statistic=`Average` · period=`60` · region=`eu-central-1`.

Boyutlar: `{"NatGatewayId": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### NAT Error Port Allocation Critical

UID: `dfxran34yedxca` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/NATGateway/ErrorPortAllocation` · statistic=`Average` · period=`60` · region=`eu-central-1`.

Boyutlar: `{"NatGatewayId": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### NAT Packets Dropped Warning

UID: `afxrc14e7h7nkf` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/NATGateway/PacketsDropCount` · statistic=`Average` · period=`60` · region=`eu-central-1`.

Boyutlar: `{"NatGatewayId": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### NAT Packets Dropped Critical

UID: `efxrd4rh449a8f` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/NATGateway/PacketsDropCount` · statistic=`Average` · period=`60` · region=`eu-central-1`.

Boyutlar: `{"NatGatewayId": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

### Outbound

Uygulamaların HTTP client çağrılarındaki hata oranı, hatalı hedef sayısı ve p95 gecikmesini izler.

**Alarm geldiğinde:** server_address bazında hedefi bul. Tempo client spanlarında DNS/connect/TLS/response sürelerini incele. Hedef servis hatasını uygulama, NAT ve DNS metrikleriyle karşılaştır; global p95 tüm çağrıların birleşik dağılımıdır.

| Kural | Severity | Durum | Gerçek condition | Pending |
|---|---|---|---|---|
| Outbound Error Rate High - Warning | warning | Duraklatılmış | B > 1 | 1m |
| Outbound Error Rate High - Critical | critical | Açık | B > 5 | 1m |
| Outbound Failing Endpoints - Warning | warning | Açık | B > 1 | 1m |
| Outbound Global p95 Latency - Warning | warning | Açık | B > 0.5 | 1m |
| Outbound Global p95 Latency - Critical | critical | Açık | B > 1 | 1m |
| Outbound Failing Endpoints - Critical | critical | Açık | B > 1 | 1m |
| Outbound Latency by Target - Warning | warning | Duraklatılmış | B > 0.5 | 1m |
| Outbound Latency by Target - Critical | critical | Duraklatılmış | B > 1 | 1m |

#### Sorgular, kapsam ve değerlendirme zincirleri

##### Outbound Error Rate High - Warning

UID: `bfxtxkgtf203kc` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
(sum(rate(http_client_request_duration_seconds_count{http_response_status_code=~"[45].."}[5m])) / sum(rate(http_client_request_duration_seconds_count[5m]))) * 100 or vector(0)
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### Outbound Error Rate High - Critical

UID: `afxtym7l5nqpsc` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
(sum(rate(http_client_request_duration_seconds_count{http_response_status_code=~"[45].."}[5m])) / sum(rate(http_client_request_duration_seconds_count[5m]))) * 100 or vector(0)
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### Outbound Failing Endpoints - Warning

UID: `bfxtzinvuhxxcf` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
count(sum(rate(http_client_request_duration_seconds_count{http_response_status_code=~"5.."}[5m])) by (server_address) > 0) or vector(0)
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### Outbound Global p95 Latency - Warning

UID: `bfxu11n464sn4c` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
histogram_quantile(0.95, sum(rate(http_client_request_duration_seconds_bucket[5m])) by (le))
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### Outbound Global p95 Latency - Critical

UID: `cfxu21c1nsk5cd` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
histogram_quantile(0.95, sum(rate(http_client_request_duration_seconds_bucket[5m])) by (le))
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### Outbound Failing Endpoints - Critical

UID: `bfxu36tyqmgowe` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
count(sum(rate(http_client_request_duration_seconds_count{http_response_status_code=~"5.."}[5m])) by (server_address) > 0) or vector(0)
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### Outbound Latency by Target - Warning

UID: `dfxu4idro3u9sa` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
histogram_quantile(0.95, sum(rate(http_client_request_duration_seconds_bucket{http_request_method="GET"}[5m])) by (le, server_address))
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### Outbound Latency by Target - Critical

UID: `ffxu5fqt8iwaoc` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
histogram_quantile(0.95, sum(rate(http_client_request_duration_seconds_bucket{http_request_method="GET"}[5m])) by (le, server_address))
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

### RDS

Aurora CPU, read/write latency karşılaştırmaları ve storage değişimini izler. Grupta üç DynamoDB kuralı da bulunur.

**Alarm geldiğinde:** era-db-prod için writer/reader kapsamını doğrula. Database Insights/Performance Insights, sorgu yükü, lock ve bağlantı sayılarına bak. Storage artışını batch/import işlemleriyle karşılaştır. DynamoDB isimli kurallar için DynamoDB kontrol adımlarını uygula.

| Kural | Severity | Durum | Gerçek condition | Pending |
|---|---|---|---|---|
| RDS Aurora CPU Utilization Warning | warning | Açık | B > 75 | 5m |
| RDS Aurora CPU Utilization Alert | critical | Açık | B > 85 | 5m |
| RDS Read Latency Warning | warning | Duraklatılmış | I > 0, 0 | 1m |
| RDS Read Latency Alert | critical | Duraklatılmış | E > 60, 0 | 1m |
| RDS Write Latency Warning | warning | Açık | E > 30, 0 | 1m |
| RDS Write Latency Warning (copy) | critical | Açık | E > 60, 0 | 1m |
| Cpu Kullanımı %80 > | critical | Açık | B > 79.9 | 1m |
| RDS DB Size Spike Alert (10 Min) | critical | Açık | D > 15, 0 | 1m |
| 🔴 Throttle (Read) | critical | Açık | B > 0.5 | 1m |
| 🔴 Throttle (Write) | critical | Açık | B > 0.5 | 1m |
| ⚠️ System Errors | critical | Açık | B > 0.5 | 1m |

#### Sorgular, kapsam ve değerlendirme zincirleri

##### RDS Aurora CPU Utilization Warning

UID: `efuyl1akok8hsb` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-21600s → now-0s.

Metrik model alanları: `AWS/RDS/CPUUtilization` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"DBClusterIdentifier": "era-db-prod"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.
Dashboard: https://grafana-new.eradev.cloud/d/aws-executive-overview-v3/aws-executive-overview?orgId=3&refresh=1m&from=1786517406057&to=1786539006057&viewPanel=9


##### RDS Aurora CPU Utilization Alert

UID: `cfuynfgr5wav4b` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-21600s → now-0s.

Metrik model alanları: `AWS/RDS/CPUUtilization` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"DBClusterIdentifier": ["era-db-prod"]}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.
Dashboard: https://grafana-new.eradev.cloud/d/aws-executive-overview-v3/aws-executive-overview?orgId=3&refresh=1m&from=1786519059979&to=1786540659979&viewPanel=9


##### RDS Read Latency Warning

UID: `cfvjzg790zj7ka` · Seçili condition: `F` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-300s → now-0s.

Metrik model alanları: `AWS/RDS/ReadLatency` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"DBClusterIdentifier": ["era-db-prod"]}`.

**B veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-7200s → now-0s.

Metrik model alanları: `AWS/RDS/ReadLatency` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"DBClusterIdentifier": ["era-db-prod"]}`.

- **C:** type=reduce, expression=`A`, reducer=mean.
- **D:** type=reduce, expression=`B`, reducer=mean.
- **G:** type=math, expression=`abs($C - $D) > 0.01`, reducer=—.
- **F:** type=threshold, expression=`I`, reducer=—.
- **H:** type=math, expression=`($C - $D) / ($D + 0.0005) * 100 > 30`, reducer=—.
- **I:** type=math, expression=`$G * $H`, reducer=—.
Dashboard: https://grafana-new.eradev.cloud/d/aws-executive-overview-v3/aws-executive-overview?orgId=3&refresh=1m&from=1787035711525&to=1787057311525&viewPanel=11


##### RDS Read Latency Alert

UID: `afvk48n4v0074f` · Seçili condition: `F` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-300s → now-0s.

Metrik model alanları: `AWS/RDS/ReadLatency` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"DBClusterIdentifier": ["era-db-prod"]}`.

**B veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-21600s → now-0s.

Metrik model alanları: `AWS/RDS/ReadLatency` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"DBClusterIdentifier": ["era-db-prod"]}`.

- **C:** type=reduce, expression=`A`, reducer=mean.
- **D:** type=reduce, expression=`B`, reducer=mean.
- **E:** type=math, expression=`(($C -$D) / $D) * 100`, reducer=—.
- **F:** type=threshold, expression=`E`, reducer=—.
Dashboard: https://grafana-new.eradev.cloud/d/aws-executive-overview-v3/aws-executive-overview?orgId=3&refresh=1m&from=1787035711525&to=1787057311525&viewPanel=11


##### RDS Write Latency Warning

UID: `efvrgeplma4n4a` · Seçili condition: `F` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-300s → now-0s.

Metrik model alanları: `AWS/RDS/WriteLatency` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"DBClusterIdentifier": ["era-db-prod"]}`.

**B veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-7200s → now-0s.

Metrik model alanları: `AWS/RDS/WriteLatency` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"DBClusterIdentifier": ["era-db-prod"]}`.

- **C:** type=reduce, expression=`A`, reducer=mean.
- **D:** type=reduce, expression=`B`, reducer=mean.
- **E:** type=math, expression=`(($C -$D) / $D) * 100`, reducer=—.
- **F:** type=threshold, expression=`E`, reducer=—.
Dashboard: https://grafana-new.eradev.cloud/d/aws-executive-overview-v3/aws-executive-overview?orgId=3&refresh=1m&from=1787035711525&to=1787057311525&viewPanel=11


##### RDS Write Latency Warning (copy)

UID: `afvrggos04e0wf` · Seçili condition: `F` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-300s → now-0s.

Metrik model alanları: `AWS/RDS/WriteLatency` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"DBClusterIdentifier": ["era-db-prod"]}`.

**B veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-7200s → now-0s.

Metrik model alanları: `AWS/RDS/WriteLatency` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"DBClusterIdentifier": ["era-db-prod"]}`.

- **C:** type=reduce, expression=`A`, reducer=mean.
- **D:** type=reduce, expression=`B`, reducer=mean.
- **E:** type=math, expression=`(($C -$D) / $D) * 100`, reducer=—.
- **F:** type=threshold, expression=`E`, reducer=—.
Dashboard: https://grafana-new.eradev.cloud/d/aws-executive-overview-v3/aws-executive-overview?orgId=3&refresh=1m&from=1787035711525&to=1787057311525&viewPanel=11


##### Cpu Kullanımı %80 >

UID: `ffvuicu9s56gwd` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

```text
SEARCH('{AWS/RDS,DBClusterIdentifier,Role} MetricName="CPUUtilization" DBClusterIdentifier="era-db-prod" Role="WRITER"', 'Average', 60)
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### RDS DB Size Spike Alert (10 Min)

UID: `dfvuozr4auepse` · Seçili condition: `E` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-1800s → now-0s.

```text
SEARCH('{AWS/RDS,DBClusterIdentifier} MetricName="VolumeBytesUsed" DBClusterIdentifier="era-db-prod"', 'Maximum', 600)
```

- **B:** type=reduce, expression=`A`, reducer=max.
- **C:** type=reduce, expression=`A`, reducer=min.
- **D:** type=math, expression=`($B - $C) / 1073741824`, reducer=—.
- **E:** type=threshold, expression=`D`, reducer=—.

##### 🔴 Throttle (Read)

UID: `bfvv62v3kb8jke` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-3600s → now-0s.

```text
SUM(SEARCH('{AWS/DynamoDB,TableName} MetricName="ReadThrottleEvents"', 'Sum', 60))
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### 🔴 Throttle (Write)

UID: `cfvv64y8huwaod` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-3600s → now-0s.

```text
SUM(SEARCH('{AWS/DynamoDB,TableName} MetricName="WriteThrottleEvents"', 'Sum', 60))
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### ⚠️ System Errors

UID: `afvv67frxts00e` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-3600s → now-0s.

```text
SUM(SEARCH('{AWS/DynamoDB,TableName} MetricName="SystemErrors"', 'Sum', 60))
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

### SNS

Bildirim teslim hatası, DLQya redrive hatası ve SMS başarı metriğini izler.

**Alarm geldiğinde:** Topic, subscription ve hedef durumunu incele. Delivery logları, subscription filter ve erişim politikalarını kontrol et. Redrive hatasında DLQ/KMS izinlerini; SMS için teslim hata kodu ve metriğin ölçeğini doğrula.

| Kural | Severity | Durum | Gerçek condition | Pending |
|---|---|---|---|---|
| SNS Notifications Failed Warning | warning | Açık | B > 0 | 5m |
| SNS Notifications Failed Critical | critical | Açık | B > 10 | 5m |
| SNS Failed To Redrive Warning | warning | Açık | B > 0 | 5m |
| SNS Failed To Redrive Critical | critical | Açık | B > 5 | 5m |
| SNS SMS Success Rate Low Warning | warning | Açık | B < 95 | 5m |
| SNS SMS Success Rate Low Critical | critical | Açık | B < 90 | 5m |

#### Sorgular, kapsam ve değerlendirme zincirleri

##### SNS Notifications Failed Warning

UID: `efxt36140r4lce` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/SNS/NumberOfNotificationsFailed` · statistic=`Sum` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"TopicName": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### SNS Notifications Failed Critical

UID: `efxt4876vgdmoe` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/SNS/NumberOfNotificationsFailed` · statistic=`Sum` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"TopicName": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### SNS Failed To Redrive Warning

UID: `bfxt5p17z2adcf` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/SNS/NumberOfNotificationsFailedToRedrive` · statistic=`Sum` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"TopicName": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### SNS Failed To Redrive Critical

UID: `efxt75gt3t3wgf` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/SNS/NumberOfNotificationsFailedToRedrive` · statistic=`Sum` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### SNS SMS Success Rate Low Warning

UID: `efxt8gi4ybr40d` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/SNS/SMSSuccessRate` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### SNS SMS Success Rate Low Critical

UID: `dfxt8sff64dmoe` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/SNS/SMSSuccessRate` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

### SQS

Queue depth, oldest message age, DLQ etiketi verilen kuyruklar ve in-flight mesajları izler.

**Alarm geldiğinde:** QueueName ve redrive policyyi doğrula. Producer/consumer hızlarını, consumer hata loglarını, visibility timeout ve oldest age trendini karşılaştır. DLQ kurallarının gerçekten DLQ kuyruklarına sınırlandığını kontrol et.

| Kural | Severity | Durum | Gerçek condition | Pending |
|---|---|---|---|---|
| SQS Queue Depth Warning | warning | Açık | B > 100 | 1m |
| SQS Queue Depth Critical | critical | Açık | B > 500 | 5m |
| SQS Oldest Message Age Warning | warning | Açık | B > 300 | 5m |
| SQS Oldest Message Age Critical | critical | Açık | B > 1800 | 5m |
| SQS DLQ Messages Critical | critical | Açık | B > 10 | 5m |
| SQS DLQ Messages Warning | warning | Açık | B > 1 | 5m |
| SQS In-Flight High Warning | warning | Açık | B > 500 | 5m |

#### Sorgular, kapsam ve değerlendirme zincirleri

##### SQS Queue Depth Warning

UID: `dfxqlypm4hhq8b` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/SQS/ApproximateNumberOfMessagesVisible` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### SQS Queue Depth Critical

UID: `ffxqmw4g7a58ga` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/SQS/ApproximateNumberOfMessagesVisible` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"QueueName": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### SQS Oldest Message Age Warning

UID: `bfxqn89qadibkb` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/SQS/ApproximateAgeOfOldestMessage` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### SQS Oldest Message Age Critical

UID: `efxqng2e0nx8ga` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/SQS/ApproximateAgeOfOldestMessage` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"QueueName": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### SQS DLQ Messages Critical

UID: `bfxqno6bjgidcc` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/SQS/ApproximateNumberOfMessagesVisible` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"QueueName": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### SQS DLQ Messages Warning

UID: `dfxqnwjdiegaof` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/SQS/ApproximateNumberOfMessagesVisible` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### SQS In-Flight High Warning

UID: `cfxqo7eif0r28a` · Seçili condition: `C` · No Data: `NoData` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/SQS/ApproximateNumberOfMessagesNotVisible` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

### WAF

BlockedRequests ve CountedRequests metriklerini izler.

**Alarm geldiğinde:** WebACL ve rule bazında sampled requests/logları incele. Engellenen IP, URI ve user agentları belirle; saldırı trafiği ile meşru istemci bloklanmasını ayır. Count metriğini bloklama olarak yorumlama.

| Kural | Severity | Durum | Gerçek condition | Pending |
|---|---|---|---|---|
| WAF Blocked Requests Spike Warning | warning | Açık | B (reduce); seçili threshold yok | 5m |
| WAF Blocked Requests Spike Critical | critical | Açık | B > 500 | 5m |
| WAF Counted Requests High Warning | warning | Açık | B > 100 | 5m |

#### Sorgular, kapsam ve değerlendirme zincirleri

##### WAF Blocked Requests Spike Warning

UID: `afxtcgt3an20wa` · Seçili condition: `B` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/WAFV2/BlockedRequests` · statistic=`Sum` · period=`300` · region=`eu-central-1`.

Boyutlar: `{"Region": "eu-central-1", "Rule": "ALL", "WebACL": "era-waf-acl"}`.

```text
{"namespace":"AWS/WAFV2","metricName":"BlockedRequests","dimensions":{},"statistics":["Sum"],"period":300,"region":"eu-central-1","refId":"A"}"}
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### WAF Blocked Requests Spike Critical

UID: `efxtdtq5pj5kwc` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/WAFV2/BlockedRequests` · statistic=`Sum` · period=`300` · region=`eu-central-1`.

Boyutlar: `{"Region": "eu-central-1", "Rule": "ALL", "WebACL": "era-waf-acl"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### WAF Counted Requests High Warning

UID: `afxtg9oeemcqob` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-600s → now-0s.

Metrik model alanları: `AWS/WAFV2/CountedRequests` · statistic=`Sum` · period=`300` · region=`eu-central-1`.

Boyutlar: `{"Rule": "ALL", "WebACL": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

### eks-prod-cluster

Node, deployment, container, storage, batch ve izleme altyapısı sağlığını izler.

**Alarm geldiğinde:** Önce ilgili kaynak adını Explore sorgusuyla belirle. kubectl describe ile Events bölümünü; container hatasında current/previous logları incele. Çok sayıda eşzamanlı veri hatasında Prometheus ve kube-state-metrics sağlığından başla.

| Kural | Severity | Durum | Gerçek condition | Pending |
|---|---|---|---|---|
| EKS PROD - Node NotReady | critical | Açık | A > 0 | 5m |
| Deployment Down - Zero Available Replicas | critical | Açık | B > 0.5 | 1m |
| EKS PROD - CrashLoopBackOff | critical | Açık | A > 0 | 5m |
| EKS PROD - Deployment Unhealthy | critical | Açık | A > 0 | 1m |
| EKS PROD - ImagePullBackOff | critical | Açık | A > 0 | 3m |
| EKS PROD - OOMKilled | critical | Açık | A > 0 | 1m |
| EKS PROD - PVC Pending | warning | Açık | A > 0 | 5m |
| EKS PROD - Failed Pods | critical | Açık | A > 0 | 5m |
| EKS PROD - Waiting Containers | warning | Açık | A > 0 | 5m |
| EKS PROD - Init Container Issues | critical | Açık | A > 0 | 5m |
| EKS PROD - Pods Pending | warning | Açık | A > 0 | 10m |
| EKS PROD - Failed Jobs | warning | Açık | A > 0 | 1m |
| EKS PROD - Evicted Pods | warning | Açık | A > 0 | 1m |
| EKS PROD - Node CPU High | critical | Açık | A > 80 | 10m |
| EKS PROD - Node Memory High (warning) | warning | Açık | A > 75 | 15m |
| EKS PROD - Pods per Node High | critical | Açık | A > 50 | 10m |
| EKS PROD - Deployment Replicas Unavailable | critical | Açık | A > 0 | 10m |
| EKS PROD - FIRING OLMALI - Watchdog (DeadMansSwitch) | tanımsız | Açık | A (query); seçili threshold yok | 0s |
| EKS PROD - kube-state-metrics Down | critical | Açık | A > 0 | 5m |
| EKS PROD - HPA Maxed Out | critical | Duraklatılmış | A > 0 | 15m |
| EKS PROD - Certificate Expiring (warning) | warning | Duraklatılmış | A < 15 | 5m |
| EKS PROD - Certificate Not Ready | critical | Duraklatılmış | A > 0 | 10m |
| EKS PROD - PVC Disk High (warning) | critical | Açık | A > 80 | 5m |
| EKS PROD - DaemonSet Unavailable | critical | Açık | A > 0 | 10m |
| EKS PROD - CoreDNS SERVFAIL High | critical | Duraklatılmış | A > 5 | 5m |

#### Sorgular, kapsam ve değerlendirme zincirleri

##### EKS PROD - Node NotReady

UID: `efuy83q5lem0we` · Seçili condition: `C` · No Data: `Alerting` · Hata: `Alerting`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
count(kube_node_info{service="obs-kube-state-metrics"})
  -
sum(kube_node_status_condition{service="obs-kube-state-metrics",condition="Ready",status="true"})
```

- **C:** type=threshold, expression=`A`, reducer=—.

##### Deployment Down - Zero Available Replicas

UID: `afuy21sdfrabkf` · Seçili condition: `C` · No Data: `Alerting` · Hata: `Error`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-3600s → now-0s.

```promql
count((kube_deployment_spec_replicas{namespace=~".*", service="obs-kube-state-metrics"} > 0) and (kube_deployment_status_replicas_available{namespace=~".*", service="obs-kube-state-metrics"} == 0)) or vector(0)
```

- **B:** type=reduce, expression=`A`, reducer=last.
- **C:** type=threshold, expression=`B`, reducer=—.

##### EKS PROD - CrashLoopBackOff

UID: `ffuycsha175dsb` · Seçili condition: `C` · No Data: `Alerting` · Hata: `Alerting`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
count(
  max by (namespace, pod, container) (
    max_over_time(kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"}[5m])
  ) > 0
) or vector(0)
```

- **C:** type=threshold, expression=`A`, reducer=—.

##### EKS PROD - Deployment Unhealthy

UID: `efuygvkmrl88we` · Seçili condition: `C` · No Data: `Alerting` · Hata: `Alerting`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
count(
  max by (namespace, deployment) (
    min_over_time(kube_deployment_status_condition{condition="Available", status="false"}[5m])
  ) > 0
) or vector(0)
```

- **C:** type=threshold, expression=`A`, reducer=—.

##### EKS PROD - ImagePullBackOff

UID: `afuyney8ozlkwc` · Seçili condition: `C` · No Data: `Alerting` · Hata: `Alerting`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
count(
  max by (namespace, pod, container) (
    max_over_time(kube_pod_container_status_waiting_reason{reason=~"ImagePullBackOff|ErrImagePull"}[5m])
  ) > 0
) or vector(0)
```

- **C:** type=threshold, expression=`A`, reducer=—.

##### EKS PROD - OOMKilled

UID: `cfuynyi182nlsb` · Seçili condition: `C` · No Data: `Alerting` · Hata: `Alerting`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
count(
  (
    max by (namespace, pod, container) (
      increase(kube_pod_container_status_restarts_total[5m])
    ) > 0
  )
  and on(namespace, pod, container) (
    max by (namespace, pod, container) (
      kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}
    ) == 1
  )
) or vector(0)
```

- **C:** type=threshold, expression=`A`, reducer=—.

##### EKS PROD - PVC Pending

UID: `cfuyoph97t340f` · Seçili condition: `C` · No Data: `Alerting` · Hata: `Alerting`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
count(
  max by (namespace, persistentvolumeclaim) (
    kube_persistentvolumeclaim_status_phase{phase="Pending"} == 1
  )
) or vector(0)
```

- **C:** type=threshold, expression=`A`, reducer=—.

##### EKS PROD - Failed Pods

UID: `bfuypbyxtxm9sd` · Seçili condition: `C` · No Data: `Alerting` · Hata: `Alerting`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
sum(
  (
    max by (namespace, pod) (
      kube_pod_status_phase{phase="Failed"} == 1
    )
  )
  unless on (namespace, pod)
  (
    max by (namespace, pod) (
      kube_pod_status_reason{reason="Evicted"} == 1
    )
  )
) or vector(0)
```

- **C:** type=threshold, expression=`A`, reducer=—.

##### EKS PROD - Waiting Containers

UID: `dfuyq5g352mm8a` · Seçili condition: `C` · No Data: `Alerting` · Hata: `Alerting`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
count(
  max by (namespace, pod, container) (
    max_over_time(kube_pod_container_status_waiting_reason{reason=~"InvalidImageName|CreateContainerConfigError|CreateContainerError|RunContainerError"}[5m])
  ) > 0
) or vector(0)
```

- **C:** type=threshold, expression=`A`, reducer=—.

##### EKS PROD - Init Container Issues

UID: `afuyratg62lmoc` · Seçili condition: `C` · No Data: `Alerting` · Hata: `Alerting`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
count(
  max by (namespace, pod, container) (
    max_over_time(kube_pod_init_container_status_waiting_reason{reason=~"CrashLoopBackOff|ImagePullBackOff|ErrImagePull|CreateContainerConfigError|CreateContainerError|RunContainerError"}[5m])
  ) > 0
) or vector(0)
```

- **C:** type=threshold, expression=`A`, reducer=—.

##### EKS PROD - Pods Pending

UID: `cfv1eps4et2iob` · Seçili condition: `C` · No Data: `Alerting` · Hata: `Alerting`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
sum(
  max by (namespace, pod) (
    kube_pod_status_phase{phase="Pending"}
  )
) or vector(0)
```

- **C:** type=threshold, expression=`A`, reducer=—.

##### EKS PROD - Failed Jobs

UID: `afv1f59eesrggc` · Seçili condition: `C` · No Data: `Alerting` · Hata: `Alerting`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
count(
  max by (namespace, job_name) (
    kube_job_status_failed
  ) > 0
) or vector(0)
```

- **C:** type=threshold, expression=`A`, reducer=—.

##### EKS PROD - Evicted Pods

UID: `dfv1fu4fqir5sd` · Seçili condition: `C` · No Data: `Alerting` · Hata: `Alerting`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
sum(
  max by (namespace, pod) (
    kube_pod_status_reason{reason="Evicted"} == 1
  )
) or vector(0)
```

- **C:** type=threshold, expression=`A`, reducer=—.

##### EKS PROD - Node CPU High

UID: `afv1iwcochekgc` · Seçili condition: `C` · No Data: `Alerting` · Hata: `Alerting`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
sum(rate(container_cpu_usage_seconds_total{container!="POD",container!=""}[5m])) by(instance)
/ on(instance) group_left()
sum(machine_cpu_cores) by(instance) * 100
```

- **C:** type=threshold, expression=`A`, reducer=—.

##### EKS PROD - Node Memory High (warning)

UID: `dfv1k7dwdnmrkf` · Seçili condition: `C` · No Data: `Alerting` · Hata: `Alerting`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
sum(container_memory_usage_bytes{container!="POD",container!=""}) by(instance)
/ on(instance) group_left()
sum(machine_memory_bytes) by(instance) * 100
```

- **C:** type=threshold, expression=`A`, reducer=—.

##### EKS PROD - Pods per Node High

UID: `cfv1lmf80yewwb` · Seçili condition: `C` · No Data: `Alerting` · Hata: `Alerting`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
sum(kubelet_running_pods) by(instance)
```

- **C:** type=threshold, expression=`A`, reducer=—.

##### EKS PROD - Deployment Replicas Unavailable

UID: `cfv1p7lhocnwgf` · Seçili condition: `C` · No Data: `Alerting` · Hata: `Alerting`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
count(
  max by (namespace, deployment) (
    kube_deployment_status_replicas_unavailable{service="obs-kube-state-metrics"}
  ) >= 1
) or vector(0)
```

- **C:** type=threshold, expression=`A`, reducer=—.

##### EKS PROD - FIRING OLMALI - Watchdog (DeadMansSwitch)

UID: `efv25abyh4s1sb` · Seçili condition: `A` · No Data: `Alerting` · Hata: `Alerting`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
vector(1)
```

- **C:** type=threshold, expression=`A`, reducer=—.

##### EKS PROD - kube-state-metrics Down

UID: `efv545d29yxogc` · Seçili condition: `C` · No Data: `OK` · Hata: `Alerting`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
absent(kube_node_info{service="obs-kube-state-metrics"})
```

- **C:** type=threshold, expression=`A`, reducer=—.

##### EKS PROD - HPA Maxed Out

UID: `ffvfw8j7xv9q8d` · Seçili condition: `C` · No Data: `Alerting` · Hata: `Alerting`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
count(
  max by (namespace, horizontalpodautoscaler) (
    kube_horizontalpodautoscaler_status_current_replicas{service="obs-kube-state-metrics"}
  )
  ==
  max by (namespace, horizontalpodautoscaler) (
    kube_horizontalpodautoscaler_spec_max_replicas{service="obs-kube-state-metrics"}
  )
) or vector(0)
```

- **C:** type=threshold, expression=`A`, reducer=—.

##### EKS PROD - Certificate Expiring (warning)

UID: `ffvgvkbx10bnkd` · Seçili condition: `C` · No Data: `Alerting` · Hata: `Alerting`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
(certmanager_certificate_expiration_timestamp_seconds - time()) / 86400
```

- **C:** type=threshold, expression=`A`, reducer=—.

##### EKS PROD - Certificate Not Ready

UID: `cfvjajrmi3pxce` · Seçili condition: `C` · No Data: `Alerting` · Hata: `Alerting`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
count(
  certmanager_certificate_ready_status{condition="True"} == 0
) or vector(0)
```

- **C:** type=threshold, expression=`A`, reducer=—.

##### EKS PROD - PVC Disk High (warning)

UID: `ffvjb16bwhudcb` · Seçili condition: `C` · No Data: `Alerting` · Hata: `Alerting`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes * 100
```

- **C:** type=threshold, expression=`A`, reducer=—.

##### EKS PROD - DaemonSet Unavailable

UID: `ffvjbzms58u80c` · Seçili condition: `C` · No Data: `Alerting` · Hata: `Alerting`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
count(
  kube_daemonset_status_number_unavailable{service="obs-kube-state-metrics"} > 0
) or vector(0)
```

- **C:** type=threshold, expression=`A`, reducer=—.

##### EKS PROD - CoreDNS SERVFAIL High

UID: `bfvjkmikfx6v4b` · Seçili condition: `C` · No Data: `OK` · Hata: `Alerting`

**A veri kaynağı:** `afl5b2ub9alfke` · Sorgu aralığı: now-600s → now-0s.

```promql
sum(rate(coredns_dns_responses_total{rcode="SERVFAIL"}[5m]))
/
sum(rate(coredns_dns_responses_total[5m]))
* 100
```

- **C:** type=threshold, expression=`A`, reducer=—.

### API-Gateway

Son 5 dakikalık 4XXError ortalamasını uzun baseline ile karşılaştırır.

**Alarm geldiğinde:** ApiName, method ve path bazında 400/401/403/404/429 dağılımını çıkar. Auth değişikliği, istemci deployu ve trafik artışını karşılaştır; baseline sıfıra yakınsa küçük artışların oranı büyütebileceğini dikkate al.

| Kural | Severity | Durum | Gerçek condition | Pending |
|---|---|---|---|---|
| API Gateway 4xx Anomaly Warning | warning | Duraklatılmış | F > 0 | 5m |
| API Gateway 4xx Anomaly Critical | critical | Duraklatılmış | F > 0 | 5m |

#### Sorgular, kapsam ve değerlendirme zincirleri

##### API Gateway 4xx Anomaly Warning

UID: `ffz1nepnx8phcd` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-300s → now-0s.

Metrik model alanları: `AWS/ApiGateway/4XXError` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"ApiName": "*"}`.

**D veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-7200s → now-0s.

Metrik model alanları: `AWS/ApiGateway/4XXError` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"ApiName": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=mean.
- **C:** type=threshold, expression=`F`, reducer=—.
- **E:** type=reduce, expression=`D`, reducer=mean.
- **F:** type=math, expression=`${B} - (${E} * 1.3)`, reducer=—.

##### API Gateway 4xx Anomaly Critical

UID: `cfz1noa6il0jkf` · Seçili condition: `C` · No Data: `OK` · Hata: `Error`

**A veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-300s → now-0s.

Metrik model alanları: `AWS/ApiGateway/4XXError` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"ApiName": "*"}`.

**D veri kaynağı:** `bfl5netc12sxsf` · Sorgu aralığı: now-7200s → now-0s.

Metrik model alanları: `AWS/ApiGateway/4XXError` · statistic=`Average` · period=`otomatik/boş` · region=`eu-central-1`.

Boyutlar: `{"ApiName": "*"}`.

- **B:** type=reduce, expression=`A`, reducer=mean.
- **C:** type=threshold, expression=`F`, reducer=—.
- **E:** type=reduce, expression=`D`, reducer=mean.
- **F:** type=math, expression=`${B} - (${E} * 1.5)`, reducer=—.

## 6. EKS için hızlı kontrol komutları

Komutlardaki yer tutucuları alarmdaki kaynakla değiştir. Önce mevcut contextin doğru cluster olduğunu doğrula.

```bash
kubectl config current-context
kubectl get nodes
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
kubectl describe node <node>
kubectl describe pod <pod> -n <namespace>
kubectl logs <pod> -n <namespace> -c <container> --tail=200
kubectl logs <pod> -n <namespace> -c <container> --previous --tail=200
kubectl get deployment <deployment> -n <namespace>
kubectl describe deployment <deployment> -n <namespace>
kubectl describe pvc <pvc> -n <namespace>
kubectl describe job <job> -n <namespace>
kubectl describe daemonset <daemonset> -n <namespace>
kubectl get hpa -A
```

Count/sum ile toplulaştırılan alarmlar kaynak etiketlerini kaybedebilir. Explore üzerinde iç sorguyu namespace/pod/deployment boyutlarıyla çalıştırarak kaynakları bul. Örnek:

```promql
max by (namespace, pod, container) (
  max_over_time(kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"}[5m])
) > 0
```

| Alarm | İlk araştırma |
|---|---|
| CrashLoop / OOMKilled | previous log, exit code, termination reason, memory limit |
| ImagePull / Waiting / Init | Events, image tag, registry izni, Secret/ConfigMap ve init logları |
| Pending / PVC Pending | CPU/RAM requests, taint/toleration, affinity, PVC/StorageClass ve CSI Events |
| Node NotReady / Evicted | node Conditions, disk/memory pressure ve kubelet Events |
| Deployment Down / Unavailable | available/desired replicas, readiness, rollout ve pod Events |
| Failed Jobs | job/pod logu, retry ve CronJob geçmişi; kalmış başarısız işler sayımı sürdürebilir |
| PVC Disk High | kullanılan/kapasite metriği, ilgili workload ve volume driverı |
| CoreDNS | DNS podları, logları ve SERVFAIL dağılımı |
| kube-state-metrics Down | exporter pod/service ve Prometheus target/scrape durumu |

## 7. DatasourceError ve bildirim davranışı

Birçok CloudWatch/Prometheus alarmında execErrState=Error bulunduğu için veri kaynağı hataları ayrı alarm durumları oluşturabilir. Bu durum gerçek metrik eşiğinin aşılmasıyla aynı değildir. İncelemede datasource bağlantısı, kimlik/yetki, sorgu sözdizimi, timeout ve scrape sağlığı kontrol edilir. Aynı anda çok sayıda alarm hata veriyorsa ortak datasource arızası öncelikli adaydır.

Export yalnızca alert rules içerir. Contact point, notification policy, mute timing, grouping, repeat interval ve inhibition davranışı bu dosyadan doğrulanamaz. Watchdog açıklamasındaki healthchecks.io yönlendirmesi ve Google Chatten hariç tutma amacı, notification policy/contact point üzerinde ayrıca kontrol edilmelidir.

## 8. Bakım ve kabul kontrolü

- Her kural için Preview/Explore ile beklenen seri, birim ve kaynak etiketleri doğrulanır.
- Duraklatılmış kurallar için kapatma sebebi ve yeniden açma kararı kayıt altına alınır.
- Başlık, severity, annotation eşiği ve gerçek condition uyumlu tutulur.
- Kritik kural için No Data/Error davranışı ve veri sağlığı izlemesi kontrol edilir.
- Bildirim kanalı ve Watchdog uçtan uca kontrollü olarak doğrulanır.
- Her eşik değişikliğinde bu kataloğun sürümü ve tarihi güncellenir.
