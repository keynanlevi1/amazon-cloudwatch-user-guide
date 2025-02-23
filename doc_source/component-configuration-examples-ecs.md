# Amazon Elastic Container Service \(Amazon ECS\)<a name="component-configuration-examples-ecs"></a>

The following example shows a component configuration in JSON format for Amazon Elastic Container Service \(Amazon ECS\)\.

```
{
    "alarmMetrics":[
       {
          "alarmMetricName":"CpuUtilized",
          "monitor":true
       },
       {
          "alarmMetricName":"MemoryUtilized",
          "monitor":true
       },
       {
          "alarmMetricName":"NetworkRxBytes",
          "monitor":true
       },
       {
          "alarmMetricName":"NetworkTxBytes",
          "monitor":true
       },
       {
          "alarmMetricName":"RunningTaskCount",
          "monitor":true
       },
       {
          "alarmMetricName":"PendingTaskCount",
          "monitor":true
       },
       {
          "alarmMetricName":"StorageReadBytes",
          "monitor":true
       },
       {
          "alarmMetricName":"StorageWriteBytes",
          "monitor":true
       }
    ],
    "logs":[
       {
          "logGroupName":"/ecs/my-task-definition",
          "logType":"APPLICATION",
          "monitor":true
       }
    ],
    "subComponents":[
       {
          "subComponentType":"AWS::ElasticLoadBalancing::LoadBalancer",
          "alarmMetrics":[
             {
                "alarmMetricName":"HTTPCode_Backend_4XX",
                "monitor":true
             },
             {
                "alarmMetricName":"HTTPCode_Backend_5XX",
                "monitor":true
             },
             {
                "alarmMetricName":"Latency",
                "monitor":true
             },
             {
                "alarmMetricName":"SurgeQueueLength",
                "monitor":true
             },
             {
                "alarmMetricName":"UnHealthyHostCount",
                "monitor":true
             }
          ]
       },
       {
          "subComponentType":"AWS::ElasticLoadBalancingV2::LoadBalancer",
          "alarmMetrics":[
             {
                "alarmMetricName":"HTTPCode_Target_4XX_Count",
                "monitor":true
             },
             {
                "alarmMetricName":"HTTPCode_Target_5XX_Count",
                "monitor":true
             },
             {
                "alarmMetricName":"TargetResponseTime",
                "monitor":true
             },
             {
                "alarmMetricName":"UnHealthyHostCount",
                "monitor":true
             }
          ]
       },
       {
          "subComponentType":"AWS::EC2::Instance",
          "alarmMetrics":[
             {
                "alarmMetricName":"CPUUtilization",
                "monitor":true
             },
             {
                "alarmMetricName":"StatusCheckFailed",
                "monitor":true
             },
             {
                "alarmMetricName":"disk_used_percent",
                "monitor":true
             },
             {
                "alarmMetricName":"mem_used_percent",
                "monitor":true
             }
          ],
          "logs":[
             {
                "logGroupName":"my_log_group",
                "logPath":"/mylog/path",
                "logType":"APPLICATION",
                "monitor":true
             }
          ],
          "processes" : [
             {
                "processName" : "my_process",
                "alarmMetrics" : [
                    {
                        "alarmMetricName" : "procstat cpu_usage",
                        "monitor" : true
                    }, {
                        "alarmMetricName" : "procstat memory_rss",
                        "monitor" : true
                    }
                 ]
             }
          ],
          "windowsEvents":[
             {
                "logGroupName":"my_log_group_2",
                "eventName":"Application",
                "eventLevels":[
                   "ERROR",
                   "WARNING",
                   "CRITICAL"
                ],
                "monitor":true
             }
          ]
       },
       {
          "subComponentType":"AWS::EC2::Volume",
          "alarmMetrics":[
             {
                "alarmMetricName":"VolumeQueueLength",
                "monitor":"true"
             },
             {
                "alarmMetricName":"VolumeThroughputPercentage",
                "monitor":"true"
             },
             {
                "alarmMetricName":"BurstBalance",
                "monitor":"true"
             }
          ]
       }
    ]
 }
```

**Note**  
The `subComponents` section of `AWS::EC2::Instance` and `AWS::EC2::Volume` applies only to Amazon ECS clusters with ECS service or ECS task running on the EC2 launch type\.
The `windowsEvents` section of `AWS::EC2::Instance` in `subComponents` applies only to Windows running on Amazon EC2 instances\.