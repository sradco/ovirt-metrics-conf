# oVirt Metrics conf
This repository stores the configuration files required to send oVirt engine, hosts and VMs metrics to a remote metrics store.

# EFK Setup

Installed the Elasticsearch , Fluentd and Kibana.

I also installed Grafana for another UI option.

Downloads: https://www.elastic.co/downloads/past-releases

* On my setup:

 - Elasticsearch  2.3.5
 - Fluentd 0.12.20
 - Kibana 4.5.4
 - Grafana

## On the hosts:

* Replaced the default /etc/collectd.conf with: [collectd.conf](https://github.com/sradco/ovirt-metrics-conf/blob/master/hosts/collectd.conf)


* Installed fluent-plugin-rewrite-tag-filter:

Run:

	Install /usr/share/gems/gems/fluentd-/bin/fluent-gem install fluent-plugin-rewrite-tag-filter


* Updated /etc/fluentd/fluent.conf. : [fluent.conf](https://github.com/sradco/ovirt-metrics-conf/blob/master/hosts/fluent.conf)


## On a central metrics store host:

Install Fluentd, rubygem-fluent-plugin-elasticsearch,  Elasticsearch and Kibana.

Updated  /etc/fluentd/fluent.conf: [Central Fluentd conf](https://github.com/sradco/ovirt-metrics-conf/blob/master/metrics-store/fluentd.conf)


In /etc/elasticsearch/elasticsearch.yml

* Appended:

	http.cors.enabled: true
	http.cors.allow-origin: "*"
	And uncomment:

network.host: "localhost"


* Uncommented in /opt/kibana/config/kibana.yml

	server.host: "localhost"


* Configured the Indexing and mapping so we can query the metrics along with the metadata.

###For metrics:

curl -XPUT 'http://localhost:9200/_template/ovirt_metrics' -d'
{
    "template": "ovirt-metrics-*",
    "order": 1,
    "settings": {
        "number_of_shards": 1 ,
        "number_of_replicas": 1
    },
    "mappings": {
        "host_statistics": {
            "_parent": {
                "type": "host_metadata"
            },
            "dynamic_templates": [
             {
                "strings": {
                    "mapping": {
                        "index": "not_analyzed",
                        "type": "string"
                    },
                    "match_mapping_type": "string",
                    "match": "*"
                }
            }
            ]
        }
    }
}'

* This will create a new index each day automatically based on the metric timestamp.


Also, will create a parent-child mapping between the metadata and the metric in order to be able to query and filter the metrics based on the metadata.

For metadata I configured the following index template:

curl -XPUT 'http://localhost:9200/_template/ovirt_metadata' -d'
{
    "template": "ovirt-metadata*",
    "order": 1,
    "settings": {
        "number_of_shards": 1,
        "number_of_replicas": 1
     },
    "mappings": {
        "vm_metadata": {
        "dynamic_templates": [
        {
            "strings": {
                "mapping": {
                    "index": "not_analyzed",
                    "type": "string"
                },
                "match_mapping_type": "string",
                "match": "*"
            }
        }
        ]
    },
    "host_metadata": {
        "dynamic_templates": [
        {
            "strings": {
                "mapping": {
                    "index": "not_analyzed",
                    "type": "string"
                },
                "match_mapping_type": "string",
                "match": "*"
            }
        }
        ]
    }
 }
}'
