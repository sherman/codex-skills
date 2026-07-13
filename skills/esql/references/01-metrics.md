# Клиент для метрик

Здесь описываем требования к клиенту для сбора метрик.

Дока актуальна для клиентов **Java**, **node.js, go, python?**

# Цели

- Упростить построение дашбоардов на основе метрик из **ELK**

# Требования

- Бизнес метрики должны писать в формате описанным тут
- Собираемые метрики:
    - **metrics.calls** - общее количество операций (могут быть успешными, фейлами и таймаутами)
    - **metrics.failures - **количество неуспешных операций
    - **metrics.duration\_max** - максимальное время выполнения операции
    - **metrics.duration\_avg\_calls** - суммарное время выполнения всех операций в нс
    - **metrics.duration\_NN\_calls** - NN перцентиль умноженное на calls
- Перcентиль считаем на стороне клиента
- Параметры сбора метрик настраиваются в **01.conf**
    - Для каждого **scope.name** настраиваем **scope.attributes.operation\_resolution**
    - Для каждого **attributes.operation\_type **настраиваем:  
- метрики которые собираем (calls/failures/timeouts/…)
    - Можно на лету поменять адрес коллектора метрик (без потери метрик)
- Гистограммы - это обычные метрики с дополнительным атрибутом - **le** (см пример ниже)
- В библиотеке надо уметь програмным путем переопределить зеленые атрибуты из спеки

# Примеры

Для примера рассмотрим **grpc client **и** custom** метрики из 01.conf админки в **prometheus** формате, и как они должны будут писаться в **ELK** нашим клиентом.

## **Prometheus формат**

```
# HELP grpc_client_processing_duration_seconds  
# TYPE grpc_client_processing_duration_seconds summary
grpc_client_processing_duration_seconds_count{method="getDiff",methodType="UNARY",service="one.tech.configuration.server.ConfigurationService",statusCode="OK"} 38204
grpc_client_processing_duration_seconds_sum{method="getDiff",methodType="UNARY",service="one.tech.configuration.server.ConfigurationService",statusCode="OK"} 245.667645499
grpc_client_processing_duration_seconds_count{method="getInitial",methodType="UNARY",service="one.tech.configuration.server.ConfigurationService",statusCode="OK"} 1
grpc_client_processing_duration_seconds_sum{method="getInitial",methodType="UNARY",service="one.tech.configuration.server.ConfigurationService",statusCode="OK"} 4.532039288
grpc_client_processing_duration_seconds_count{method="subscribe",methodType="BIDI_STREAMING",service="one.tech.configuration.server.ConfigurationService",statusCode="CANCELLED"} 2
grpc_client_processing_duration_seconds_sum{method="subscribe",methodType="BIDI_STREAMING",service="one.tech.configuration.server.ConfigurationService",statusCode="CANCELLED"} 207.202965225
grpc_client_processing_duration_seconds_count{method="subscribe",methodType="BIDI_STREAMING",service="one.tech.configuration.server.ConfigurationService",statusCode="OK"} 3014
grpc_client_processing_duration_seconds_sum{method="subscribe",methodType="BIDI_STREAMING",service="one.tech.configuration.server.ConfigurationService",statusCode="OK"} 904221.196058175
grpc_client_processing_duration_seconds_count{method="subscribe",methodType="BIDI_STREAMING",service="one.tech.configuration.server.ConfigurationService",statusCode="UNAVAILABLE"} 2
grpc_client_processing_duration_seconds_sum{method="subscribe",methodType="BIDI_STREAMING",service="one.tech.configuration.server.ConfigurationService",statusCode="UNAVAILABLE"} 238.492608935
# HELP grpc_client_processing_duration_seconds_max  
# TYPE grpc_client_processing_duration_seconds_max gauge
grpc_client_processing_duration_seconds_max{method="getDiff",methodType="UNARY",service="one.tech.configuration.server.ConfigurationService",statusCode="OK"} 0.01465738
grpc_client_processing_duration_seconds_max{method="getInitial",methodType="UNARY",service="one.tech.configuration.server.ConfigurationService",statusCode="OK"} 0.0
grpc_client_processing_duration_seconds_max{method="subscribe",methodType="BIDI_STREAMING",service="one.tech.configuration.server.ConfigurationService",statusCode="CANCELLED"} 0.0
grpc_client_processing_duration_seconds_max{method="subscribe",methodType="BIDI_STREAMING",service="one.tech.configuration.server.ConfigurationService",statusCode="OK"} 300.010241026
grpc_client_processing_duration_seconds_max{method="subscribe",methodType="BIDI_STREAMING",service="one.tech.configuration.server.ConfigurationService",statusCode="UNAVAILABLE"} 0.0

# TYPE custom method_call_duration_micrometer_seconds histogram
method_call_duration_micrometer_seconds_bucket{method_name="GetCount",le="0.1"} 18
method_call_duration_micrometer_seconds_bucket{method_name="GetCount",le="0.5"} 106
method_call_duration_micrometer_seconds_bucket{method_name="GetCount",le="1.0"} 201
method_call_duration_micrometer_seconds_bucket{method_name="GetCount",le="+Inf"} 201
method_call_duration_micrometer_seconds_count{method_name="GetCount"} 201
method_call_duration_micrometer_seconds_sum{method_name="GetCount"} 96.62
```

## **01.metrics** формат

grpc метрики продолжают писаться в таком же виде

```
# HELP grpc_client_processing_duration_seconds  
# TYPE grpc_client_processing_duration_seconds summary
grpc_client_processing_duration_seconds_count{method="getDiff",methodType="UNARY",service="one.tech.configuration.server.ConfigurationService",statusCode="OK"} 38204
grpc_client_processing_duration_seconds_sum{method="getDiff",methodType="UNARY",service="one.tech.configuration.server.ConfigurationService",statusCode="OK"} 245.667645499
grpc_client_processing_duration_seconds_count{method="getInitial",methodType="UNARY",service="one.tech.configuration.server.ConfigurationService",statusCode="OK"} 1
grpc_client_processing_duration_seconds_sum{method="getInitial",methodType="UNARY",service="one.tech.configuration.server.ConfigurationService",statusCode="OK"} 4.532039288
grpc_client_processing_duration_seconds_count{method="subscribe",methodType="BIDI_STREAMING",service="one.tech.configuration.server.ConfigurationService",statusCode="CANCELLED"} 2
grpc_client_processing_duration_seconds_sum{method="subscribe",methodType="BIDI_STREAMING",service="one.tech.configuration.server.ConfigurationService",statusCode="CANCELLED"} 207.202965225
grpc_client_processing_duration_seconds_count{method="subscribe",methodType="BIDI_STREAMING",service="one.tech.configuration.server.ConfigurationService",statusCode="OK"} 3014
grpc_client_processing_duration_seconds_sum{method="subscribe",methodType="BIDI_STREAMING",service="one.tech.configuration.server.ConfigurationService",statusCode="OK"} 904221.196058175
grpc_client_processing_duration_seconds_count{method="subscribe",methodType="BIDI_STREAMING",service="one.tech.configuration.server.ConfigurationService",statusCode="UNAVAILABLE"} 2
grpc_client_processing_duration_seconds_sum{method="subscribe",methodType="BIDI_STREAMING",service="one.tech.configuration.server.ConfigurationService",statusCode="UNAVAILABLE"} 238.492608935
# HELP grpc_client_processing_duration_seconds_max  
# TYPE grpc_client_processing_duration_seconds_max gauge
grpc_client_processing_duration_seconds_max{method="getDiff",methodType="UNARY",service="one.tech.configuration.server.ConfigurationService",statusCode="OK"} 0.01465738
grpc_client_processing_duration_seconds_max{method="getInitial",methodType="UNARY",service="one.tech.configuration.server.ConfigurationService",statusCode="OK"} 0.0
grpc_client_processing_duration_seconds_max{method="subscribe",methodType="BIDI_STREAMING",service="one.tech.configuration.server.ConfigurationService",statusCode="CANCELLED"} 0.0
grpc_client_processing_duration_seconds_max{method="subscribe",methodType="BIDI_STREAMING",service="one.tech.configuration.server.ConfigurationService",statusCode="OK"} 300.010241026
grpc_client_processing_duration_seconds_max{method="subscribe",methodType="BIDI_STREAMING",service="one.tech.configuration.server.ConfigurationService",statusCode="UNAVAILABLE"} 0.0
```

custom метрики пишутся в операционном формате

```
scope.name = 01-metrics/custom
attributes.operation_type = 01.conf.admin

Атрибуты:
attributes.operation = method_call_duration_micrometer
attributes.method_name
attributes.le

Метрики для всех операций:
metrics.calls - количество всех вызовов в рамках бакета
metrics.duration_max - максимальное время выполнения за период аггрегации
metrics.duration_avg_calls - суммарное время выполнения всех операций в нс
metrics.duration_NN_calls - NN перцентиль умноженное на calls в нс
```

# Детали реализации

Назовем наш формат “операционным”

Итого получается два варианта реализации

1. Операционный вид поддерживаем только для бизнес метрик - метрики которые разработчики собирают сами через наш API
2. Сбор метрик в операционном виде для JVM/Tomcat/REST API/grpc… делаем самостоятельно, а то до чего не дотянуться оставляем в том виде в котором есть
3. Для тех метрик которые не смогли переделать на операционный вид - делаем дашбоарды в кибане. В **charts** для них делаем отдельную поддержку.  
Что такое отдельная поддержка в **charts**:  какой-то ручной маппинг метрик на **operation.** 

# API клиента

Здесь опишем ожидаемый вид АПИ для операционных метрик.

Оснвоные составляющие операционных метрик:

- operation\_namespace - набор метрик с одинаковыми атрибутами/тегами
- operation - первый атрибут/тег, который несет основную смысловую нагрузку метрики
- params - второй и последующий атрибуты/теги

Для чего нам надо объединять метрики в namespace: для того чтобы в дальнейшем проще  построить дашбоард для метрик. 

Если разработчик решил добавить новый параметр в namespace, ему необходимо добавить его в коде дополнительным парамтером и прописать его название в 01.conf

## Общий АПИ

```java
interface OperationMetrics { 
    //Возвращает время начала операции в нс
    long startOperation();

    //операция success c duration=0
    void success(String operation, String key1, String value1); 
    void success(String operation, String key1, String value1, String key2, String value2); 
    //Здесь пример приведен для Java библиотеки, есть набор методов с явными параметрами, чтобы быть gc-friendly
    //Для случаев когда необходимо более 4х аттрибутов, предлагается такой АПИ:
    void success(String operation, Tag... tags); 

    //операция success, которая считаей duration начиная от startTime
    void success(long startTime, String operation, String key1, String value1); 
    void success(long startTime, String operation, String key1, String value1, String key2, String value2); 
    void success(long startTime, String operation, Tag... tags); 
  
    //операция success, увеливиющая счетчик сразу на count
    void successCount(int count, String operation, String key1, String value1); 
    //операция success, принимающая уже расчитанный duration
    void successDuration(long duration, String operation, String key1, String value1); 

    //операция failure c duration=0
    void failure(String operation, String key1, String value1); 
    //операция failure, которая считаей duration начиная от startTime
    void failure(long startTime, String operation, String key1, String value1); 
    //операция failure, увеливиющая счетчик сразу на count
    void failureCount(int count, String operation, String key1, String value1); 
    //операция failure, принимающая уже расчитанный duration
    void failureDuration(long duration, String operation, String key1, String value1);   
}

//интерфейс OperationMetrics получаем через вызов
OperationMetrics operationMetrics = OneMetricsFactory.getOperationMetrics(namespace) 
```

## Конфигурация метрик в 01.conf

Общая конфигурация (пропертя **global.configuratuin**):

```
{
  maxMetricsLimit = 1000 # максимальный лимит для операционных метрик по всем группам
  maxIdleTimeSec  = 60 # максимальное время между обновлением метрик, этот параметр используется для того, чтобы удалять неиспользуемые метрики из памяти
}
```


Все метрики находится в приложении one-metrics.

Конфигурация для namespace задается как: operation\_namespace=config ([hocon format](https://github.com/lightbend/config))

Имя свойства в конфиге - это имя operation\_namespace.

01.conf/01-conf-metrics

```none
# 01-conf-metrics - это часть scope.name после слеша scope.name=01-metrics/01-conf-metrics
type = "operation" # по типу определяем механизм сбора метрик
operationType = "01-conf-admin" # необязательный парамтер, задает дополнительный разрез в группу парамтеров
trace = true # включает вывод отправляемых метрик в лог (каждый период аггрегации)
disabled = false # можно отключить сбор метрик  
percentiles {
    enabled = true # включает сбор персентилей, по умолчанию отключены
    type = "OK" # OK or DATA_DOG
    values = [0.5, 0.999] # список персентилей, по дефолту: 0.5, 0.9, 0.99
}
attributes {
    custom_param_1 { # задаем название атрибутов, в примере ниже это "custom-param-value1"
        default = "na" # возможность задать значение по умолчанию, когда тег не передали
    }
    custom_param_2 { # задаем название атрибутов, в примере ниже это "100"
        default = "na" # возможность задать значение по умолчанию, когда тег не передали
    }
}
```

С помощью **type **задаем механизм сбора метрик, здесь привел не все возможные типы, а те которые точно будем делать первым делом:

- operation - обычные операционные метрики которые описаны в начале документа
- jmx - сбор метрик из JMX бинов и публикация их в операционном виде, у этого коллектора будет свой набор параметров (задаем методы JMX бина и соотвествующий ему аттрибут - TODO)
- Только перечисленные в конфигурации конкретной группы атрибуты (attributes), должны быть переданны с операционными метриками данной группы
- Если в коде передали атрибут, которого нет в конфигурации, он должен быть проигнорирован
- Если в коде НЕ передали атрибут, который есть в конфигурации, то берется, указанное значение, по-умолчанию. Если и оно не задано, тогда берется значение: “na”.

## Пример в Java коде

```java
class MyService {
  private final OperationMetrics operationMetrics;

  public MyService(OneMetricsFactory oneMetricsFactory) {
    this.operationMetrics = oneMetricsFactory.getOperationMetrics("01-conf-metrics");
  }

  public void businesLogic() {
    long startTime = operationMetrics.startOperation();
    try {
      //some logic
      operationMetrics.success("busines-logic-1", startTime,  "custom-param-key1", "custom-param-value1", 
                                                              "custom-param-key2", "100");
    } catch(CustomException e) {
      operationMetrics.failure("busines-logic-1", startTime,  "custom-param-key1", "custom-param-value1", 
                                                              "custom-param-key2", "100");
    }
  }
}
```

## Предопределенные метрики

Так же стоит сделать набор предопределенных метрик, для которых уже есть конфигурации, и разработчик может их использовать как есть без дополнительной конфигурации, например

### API

Например мы делаем что-то такое, и добавляем в 01.conf необходимую конфигурацию

```java
class ApiMetrics {
  private final static String ENDPOINT = "endpoint";
  private final static String TENANT_ID = "tenant-id";
  private final static String METHOD = "method";
  private final static String PARAM = "param";
  
  private final OperationMetrics operationMetrics;

  public GrpcMetrics(OneMetricsFactory oneMetricsFactory) {
    this.operationMetrics = oneMetricsFactory.getOperationMetrics("api-metrics");
  }

  public void callSuccess(String endpoint, Long tenantId, String method, String param) {
    operationMetrics.success("call", ENDPOINT, endpoint,
                                     TENANT_ID, tenantId, 
                                     METHOD, method,
                                     PARAM, param)
  }
  
  public void callFailure(String endpoint, Long tenantId, String method, String param) {
    operationMetrics.failure("call", ENDPOINT, endpoint,
                                     TENANT_ID, tenantId, 
                                     METHOD, method,
                                     PARAM, param)
  }
}
```

В каком-то сервисе разработчик пишет:

```java
class MyController {
  private final ApiMetrics apiMetrics;

  public MyService(ApiMetrics apiMetrics) {
    this.apiMetrics = apiMetrics;
  }

  public void method1(String param) {
    try {
      //some logic
      apiMetrics.success("endpoint1", TenantContext.getTenantId(), "method1", param);
    } catch(CustomException e) {
      apiMetrics.failure("endpoint1", TenantContext.getTenantId(), "method1", param);
    }
  }
}
```

# Спецификация формата уровня OTEL протокола

В этой секции описанно то, как операционные метрики должны экспортироваться в open telemetry.

- Все атрибуты (resource, scope, metrics и другие) должны быть типа string
- scope.name имеет тип string
- scope.name начинается с префикса `01-metrics/`
- unit всегда равен значению units
- gauge в SDK транслируется в метрику calls:
    - calls всегда 1
    - duration\_avg\_calls, duration\_min, duration\_max - фактическое значение

| **метрика** | **тип метрики** | **тип точки** | **примечание** |
| --- | --- | --- | --- |
| calls | sum | целое (asInt) |  |
| duration\_avg\_calls | gauge | целое (asInt) |  |
| duration\_min | gauge | целое (asInt) |  |
| duration\_max | gauge | целое (asInt) |  |
| duration\_XX\_calls | gauge | целое (asInt) |  |
