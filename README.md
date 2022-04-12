# docker-swarm-hot-warm

#### Thực hiện cấu hình swarm

Cụm swarm gồm 3 máy

Trên máy manager:

```
docker swarm init --advertise-addr [ip manager]
```

Dùng token trên, qua 2 máy worker xin gia nhập cụm swarm.

Trên các máy đều được cài docker. Thực hiện thiết lập cho vm.max_map_count

```
sudo sysctl -w vm.max_map_count=262144
```

Build swarm với file docker-compose.yml

```
docker stack deploy -c docker-compose.yml [name]
```

Thực hiện thiết lập gán node trên các worker trên swarm:

```
docker node update --label-add lbes01==true manager
docker node update --label-add lbes02==true worker1
docker node update --label-add lbes03==true worker2
```

Thực hiện run file sh.

```
sh setup/setup.sh
```

Nếu không thực hiện được file sh thì thực hiện put trên dev-tools của kibana.

### Access Elasticsearch 

http://localhost:9200

### Access Kibana

http://localhost:5601

#### Setup cluster - with Kibana

##### Index Lifecycle Management (ILM)

1. Thiết lập kiểm tra ILM

- Theo mặc đinh, thời gian kiểm tra ILM là 10 phút. Thay đổi thời gian thành 5s để dễ theo dõi sự thay đổi của index giữa các node.

```
PUT _cluster/settings
{
  "persistent": {
    "indices.lifecycle.poll_interval": "5s",
    "slm.retention_schedule": "* * * * * ?"
  }
}
```

2. Thực hiện tạo policy

```
PUT /_ilm/policy/logs-hot-warm
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "1gb",
            "max_docs": "10000"
          },
          "set_priority": {
            "priority": 100
          }
        }
      },
      "warm": {
        "min_age": "30d",
        "actions": {
          "readonly" : { },
          "allocate" : {
            "number_of_replicas" : 0,
            "require" : {
              "box_type": "warm"
            }
          },
          "set_priority": {
            "priority": 50
          }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

* Tạo policy logs-hot-warm. 

	* Hot: Index đang được cập nhật và truy vấn.
	* Warm: Index không được cập nhật nhưng vẫn được truy vấn.

* Các thuộc tính của policy:
	* Rollover:
		* max_age: thời gian tối đa kể từ khi tạo index để bắt đầu cuộn qua index.
		* max_docs: số documents tối đa của index, không tinh ở các bản replicas.
		* max_size: kích thước tối đa của index, là tổng kích thước các shards của index, không tính trên các bản sao.
		* max_primary_shard_size: kich thước tối dad của shard chính.
		* min_age: thời gian tối thiểu đê bắt đầu chuyển index.
	* Priority: độ ưu tiên, độ ưu tiên càng lớn thì được thực hiện trước.

3. Tạo template cho index

```
PUT _template/template_logs
{
  "index_patterns": ["testlog-*"],
  "settings": {
    "index.number_of_shards" : 1,
    "index.number_of_replicas" : 1,
    "lifecycle.name": "logs-hot-warm",
    "lifecycle.rollover_alias": "test-logs"   
  },
  "mappings": {
    "properties" : {
      "name": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "age":{
          "type":"long"
      }
    }
  }
}
```

4.  Khởi tạo index

```
PUT testlog -000001
{
  "aliases": {
    "test-logs":{
      "is_write_index": true
    }
  }
}
```

#### Watch the generations - Kibana line
- Xem các roles trên node.
	* m: master node
	* s: content tier
	* h: hot data tier
	* w: warm data tier
	* c: cold data tier
	* f: frozen data tier

```
GET _cat/nodes?v&h=name,node.role&s=name
```

```
// kiểm tra node
GET _cat/nodeattrs?v&h=node,attr,value&s=attr:desc
GET _cat/thread_pool/search_throttled?v&h=node_name,name,active,rejected,queue,completed&s=node_name
// kiểm tra index trên node
GET _cat/shards/testlog-000001?v&h=index,shard,prirep,node&s=node
// xem chi tiết index
GET .testlog-*/_ilm/explain
GET testlog-000001/_ilm/explain
GET /_cat/indices?v
GET _cat/indices/testlog-000001?v&h=health,status,index,pri,rep,docs.count,store.size
```




