# Microservices Logs Pre-processing Pipelines
----

This pack contains a set of **pre-processing** pipelines for sources dedicated to microservices logs. This includes popular engines including Docker, Kubernetes, and Pivotal Cloud Foundry (PCF). Furthermore, there are pipelines for both JSON format and CRIO/Containerd formatted logs. Lastly, there are also pipelines for multi-line logs.

## List of pipelines and what they do
- **JSON_single_line_events** Use as preferred pipeline for single line JSON events. This uses a parser function and minimal regex for optimized performance
- **JSON_Multiline_via_Regex-Extract** Use when you know there are multi-line events in your logs. Ideally, you have a dedicated HTTP token or source for this dataset to miminize regular expressions and maximize throughput. Multiline logs might look like this:
```
{"log": "2021-02-09 11:19:04.816 ERROR [nio-8080-exec-4] x.x.x.exceptions.RestExceptionHandler: 422 Status Code - EntityNotFoundException - XXXXXXXXXXXXXXXXXXXXXXXXX", "stream": "stderr", "time": "2020-10-05T00:00:30.082640485Z"}
{"log": "xxxx.xxx.microservices.exceptions.EntityNotFoundException: XXXXXXXXXXXXXXXXXXXXXXXXX", "stream": "stderr", "time": "2020-10-05T00:00:30.082640485Z"}
{"log": "   at xxx.xxx.microservices.service.XXXXXXX.lambda$getXXXData$3(XYXYXYXYX.java:139) ~[classes!/:na]", "stream": "stderr", "time": "2020-10-05T00:00:30.082640485Z"}
{"log": "   at java.base/java.util.Optional.orElseThrow(Optional.java:408) ~[na:na]", "stream": "stderr", "time": "2020-10-05T00:00:30.082640485Z"}
```
- **JSON_Universal** Alternative pipeline to the single-line and multi-line policies. This is heavy with regular expression, so use cautiously
- **PCF_Universal** Universal pipeline for all single-line PCF events
- **CRI_Universal** Universal pipeline for single-line or multi-line logs with CRIO format similar to this:
```
2021-02-09T11:19:04.815514933+00:00 stdout F 2021-02-09 11:19:04.814 DEBUG 1 --- [nio-8080-exec-8] x.x.x.service.DocumentService            : retrieved: []
2021-02-09T11:19:04.817387066+00:00 stdout P 2021-02-09 11:19:04.816 ERROR 1 --- [nio-8080-exec-4] x.x.x.exceptions.RestExceptionHandler    : 422 Status Code - EntityNotFoundException - XXXXXXXXXXXXXXXXXXXXXXXXX
2021-02-09T11:19:04.817387066+00:00 stdout P xxx.xxx.microservices.exceptions.EntityNotFoundException: XXXXXXXXXXXXXXXXXXXXXXXXX
2021-02-09T11:19:04.817387066+00:00 stdout F    at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:190) ~[spring-web-5.2.7.RELEASE.jar!/:5.2.7.RELEASE]
```

## Requirements

Please be sure to perform the following:

* Assign this pack as the pre-processing pipeline for any LogStream sources that process microservices logs. This can include Splunk HEC, Elastic, HTTP, TCP JSON. 
* As a best practice, create different tokens for different known datasets 
  - (e.g. different token for containerized application generating web access logs, different token for applications using log4j, etc.) 
  - This will help with simpler downstream filtering to apply the correct pipelines without repeated regular expressions.
* At the source level, also create the following metadata fields to simplify downstream processing. Fields: 
 1.  **containerization_engine**. Assign it values such as **'kubernetes'**, **'k8s_crio'**, or **'docker_json'**
 2.  **multiline**. Assign this to the value: **_raw.regmatch('\n.*\n') ? true : false**
 3.  Load event breaker rules for multi-line logs and to ensure the correct timestamp is processed. See next section.


## Loading Event Breaker rules
Add the below event breaker rules:
- Go to **Knowledge | Event Breaker Rules | Add New | Advanced Mode**
- Copy the below JSON and paste into the newly created rule. 
- Apply one of these rules when capturing incoming data at the source to have it correctly parsed
- Modify the event breaker rule to suit your timestamp format or deviations in your log format

**Ruleset to import**
```
{
  "id": "Multiline-TimestampAtBeginning",
  "lib": "custom",
  "rules": [
    {
      "condition": "_raw.match(`\"(log|message|msg|line)\":`)",
      "type": "regex",
      "timestampAnchorRegex": "/(log|message|msg|line)/",
      "timestamp": {
        "type": "auto",
        "length": 150
      },
      "timestampTimezone": "local",
      "timestampEarliest": "-420weeks",
      "timestampLatest": "+1week",
      "maxEventBytes": 51200,
      "disabled": false,
      "eventBreakerRegex": "/(?=\"(?:log|message|msg|line)\":\\s*\"\\s*\\d{4}-\\d{2}-\\d{2})/m",
      "name": "JSONFileFormat",
      "fields": [
        {
          "name": "multiline",
          "value": "/\\n.*\\n/.test(_raw) ? true : false"
        }
      ]
    },
    {
      "condition": "_raw.match(`[\\d\\-T:.+]+\\s+\\w+\\s+[FP]+`)",
      "type": "regex",
      "timestampAnchorRegex": "/^[\\d\\-T:.+]+\\s+\\w+\\s+[FP]\\s+/",
      "timestamp": {
        "type": "auto",
        "length": 150
      },
      "timestampTimezone": "local",
      "timestampEarliest": "-420weeks",
      "timestampLatest": "+1week",
      "maxEventBytes": 51200,
      "disabled": false,
      "eventBreakerRegex": "/^(?=[\\d\\-T:.+]+\\s+\\w+\\s+[FP]+\\s+\\d{4}-\\d{2}-\\d{2})/m",
      "fields": [
        {
          "value": "/\\n.*\\n/.test(_raw) ? true : false",
          "name": "multiline"
        }
      ],
      "name": "RFC3339Nano "
    },
    {
      "condition": "/^\\d{2}\\s+\\w{3}\\s+\\d{4}\\s+\\d{2}:\\d{2}:\\d{2}\\s+\\w+\\s+\\[[^\\]]+\\]/.test(_raw) || sourcetype=='java' || sourcetype=='tomcat' || sourcetype=='catalina.out'",
      "type": "regex",
      "timestampAnchorRegex": "/^/",
      "timestamp": {
        "type": "auto",
        "length": 150,
        "format": "%d\\s%b\\s%Y %H:%M:%S"
      },
      "timestampTimezone": "local",
      "timestampEarliest": "-1420weeks",
      "timestampLatest": "+1week",
      "maxEventBytes": 51200,
      "disabled": false,
      "eventBreakerRegex": "/(?=^\\d{2}\\s+\\w{3}\\s+\\d{4}\\s+\\d{2}:\\d{2}:\\d{2})/gm",
      "name": "Multline_Timestamp_atbeginning",
      "fields": [
        {
          "name": "multiline",
          "value": "/\\n.*\\n/.test(_raw) ? true : false"
        }
      ]
    }
  ],
  "description": "Multiline log formats"
}
```
**End of ruleset to import**  
__Note: subsequent releases of this Pack will include the event breaker rules when it's supported in the framework__

## Release Notes

### Version 0.5 - 2021-06-28
First release. 
Pipelines for JSON & CRIO formats from docker, kubernetes, containerd, and crio. Includes pipelines for multi-line events
Pipelines for Pivotal Cloud Foundry

## Contributing to the Pack
---
Discuss this pack on our Community Slack channel [#packs](https://cribl-community.slack.com/archives/C021UP7ETM3).

## Contact
---
The author of this pack is Ahmed Kira and can be contacted at <akira@cribl.io>.

## License
---
This Pack uses the following license: [`Apache 2.0`](https://github.com/criblio/appscope/blob/master/LICENSE).