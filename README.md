# elastic-bigdata-search-analytics-cluster
Recommendations for creating and benchmarking big data search analytics cluster based on Elasticsearch

<br/> 1. Install Elasticsearch on all nodes and start service without changing config, just to make sure everything is up and running: <br/> ```service elasticsearch start```  <br/> ```service elasticsearch status```
 <br/> ```curl localhost:9200```
 
 <br/> 2. After that stop the services and configure the elasticsearch.yml usually found at /etc/elasticsearch/elasticsearch.yml in linux machines, following config can be a good start: <br/>
```
#give your cluster a name.

cluster.name: my-cluster

#give your nodes a name (change node number from node to node).
node.name: "es-node-1"

#define node 1 as master-eligible:
node.master: true

#define nodes 2 and 3 as data nodes:
node.data: true

#enter the private IP and port of your node:
network.host: 172.11.61.27
http.port: 9200



#detail the private IPs of your nodes:
discovery.zen.ping.unicast.hosts: ["172.11.61.27", "172.31.22.131","172.31.32.221"] #private IPs
```
 <br/> 3. Also to avoid <strong>split-brain</strong> situation we need to define atleast two master nodes. A “split-brain” situation is when communication between nodes in the cluster fails due to either a network failure or an internal failure with one of the nodes. In this kind of scenario, more than one node might believe it is the master node, leading to a state of data inconsistency. 

```discovery.zen.minimum_master_nodes: 2 ```

<br/> 4. Next is assigning the max memory for each node that will handle, a rule of thumb is to assign 50% of memory of your machine but no more than 32 GB. Another factor that'd contribute to your situation is your usecase. Generally in search usecases it's a good practice to allocate 1 GB of memory for every 4 GB of data on disk. This would also define the number of nodes you need for your usecase. For this goto the following file, I'm setting memory to 16 GBs.
```
sudo vim /etc/elasticsearch/jvm.options

-Xms16g
-Xmx16g
```
<br/> 5. Turn off the firewall, this may lead to not letting your node join the cluster.
```
service iptables stop
```
<br/> 6. Increase file descriptors and CPU prcsesses limit by adding the following lines to following file:
```sudo vim  /etc/security/limits.conf
  - nofile 65536
  - nproc 65536
  ```
 <br/> 7. Disable memory swapping:
```
sudo vim /etc/elasticsearch/elasticsearch.yml
bootstrap.mlockall: true

sudo vim /etc/default/elasticsearch
MAX_LOCKED_MEMORY=unlimited
```
 <br/> 8. Increase virtual memory limit to avoid hitting the limits:
 ```
 sudo vim /etc/sysctl.conf
 vm.max_map_count=262144
 ```
 ## Benchmarking the limits by using a python script:
 <br/> 1. Use the following script and generate around 400-500 million documents if you are expecting over 50 GB of data :
 ```
 #!/usr/bin/python

import json
import time
import logging
import random
import string
import uuid
import datetime

import tornado.gen
import tornado.httpclient
import tornado.ioloop
import tornado.options

try:
    xrange
    range = xrange
except NameError:
    pass

async_http_client = tornado.httpclient.AsyncHTTPClient()
headers = tornado.httputil.HTTPHeaders({"content-type": "application/json"})
id_counter = 0
upload_data_count = 0
_dict_data = None



def delete_index(idx_name):
    try:
        url = "%s/%s?refresh=true" % (tornado.options.options.es_url, idx_name)
        request = tornado.httpclient.HTTPRequest(url, headers=headers, method="DELETE", request_timeout=240, auth_username=tornado.options.options.username, auth_password=tornado.options.options.password, validate_cert=tornado.options.options.validate_cert)
        response = tornado.httpclient.HTTPClient().fetch(request)
        logging.info('Deleting index  "%s" done   %s' % (idx_name, response.body))
    except tornado.httpclient.HTTPError:
        pass


def create_index(idx_name):
    schema = {
        "settings": {
            "number_of_shards":   tornado.options.options.num_of_shards,
            "number_of_replicas": tornado.options.options.num_of_replicas
        },
        "refresh": True
    }

    body = json.dumps(schema)
    url = "%s/%s" % (tornado.options.options.es_url, idx_name)
    try:
        logging.info('Trying to create index %s' % (url))
        request = tornado.httpclient.HTTPRequest(url, headers=headers, method="PUT", body=body, request_timeout=240, auth_username=tornado.options.options.username, auth_password=tornado.options.options.password, validate_cert=tornado.options.options.validate_cert)
        response = tornado.httpclient.HTTPClient().fetch(request)
        logging.info('Creating index "%s" done   %s' % (idx_name, response.body))
    except tornado.httpclient.HTTPError:
        logging.info('Looks like the index exists already')
        pass


@tornado.gen.coroutine
def upload_batch(upload_data_txt):
    try:
        request = tornado.httpclient.HTTPRequest(tornado.options.options.es_url + "/_bulk",
                                                 method="POST",
                                                 body=upload_data_txt,
                                                 headers=headers,
                                                 request_timeout=tornado.options.options.http_upload_timeout,
                                                 auth_username=tornado.options.options.username, auth_password=tornado.options.options.password, validate_cert=tornado.options.options.validate_cert)
        response = yield async_http_client.fetch(request)
    except Exception as ex:
        logging.error("upload failed, error: %s" % ex)
        return

    result = json.loads(response.body.decode('utf-8'))
    res_txt = "OK" if not result['errors'] else "FAILED"
    took = int(result['took'])
    logging.info("Upload: %s - upload took: %5dms, total docs uploaded: %7d" % (res_txt, took, upload_data_count))


def get_data_for_format(format):
    split_f = format.split(":")
    if not split_f:
        return None, None

    field_name = split_f[0]
    field_type = split_f[1]

    return_val = ''

    if field_type == "bool":
        return_val = random.choice([True, False])

    elif field_type == "str":
        min = 3 if len(split_f) < 3 else int(split_f[2])
        max = min + 7 if len(split_f) < 4 else int(split_f[3])
        length = generate_count(min, max)
        return_val = "".join([random.choice(string.ascii_letters + string.digits) for x in range(length)])

    elif field_type == "int":
        min = 0 if len(split_f) < 3 else int(split_f[2])
        max = min + 100000 if len(split_f) < 4 else int(split_f[3])
        return_val = generate_count(min, max)
    
    elif field_type == "ipv4":
        return_val = "{0}.{1}.{2}.{3}".format(generate_count(0, 245),generate_count(0, 245),generate_count(0, 245),generate_count(0, 245))

    elif field_type in ["ts", "tstxt"]:
        now = int(time.time())
        per_day = 24 * 60 * 60
        min = now - 30 * per_day if len(split_f) < 3 else int(split_f[2])
        max = now + 30 * per_day if len(split_f) < 4 else int(split_f[3])
        ts = generate_count(min, max)
        return_val = int(ts * 1000) if field_type == "ts" else datetime.datetime.fromtimestamp(ts).strftime("%Y-%m-%dT%H:%M:%S.000-0000")

    elif field_type == "words":
        min = 2 if len(split_f) < 3 else int(split_f[2])
        max = min + 8 if len(split_f) < 4 else int(split_f[3])
        count = generate_count(min, max)
        words = []
        for _ in range(count):
            word_len = random.randrange(3, 10)
            words.append("".join([random.choice(string.ascii_letters + string.digits) for x in range(word_len)]))
        return_val = " ".join(words)

    elif field_type == "dict":
        global _dict_data
        min = 2 if len(split_f) < 3 else int(split_f[2])
        max = min + 8 if len(split_f) < 4 else int(split_f[3])
        count = generate_count(min, max)
        return_val = " ".join([random.choice(_dict_data).strip() for _ in range(count)])

    elif field_type == "text":
        text = ["text1", "text2", "text3"] if len(split_f) < 3 else split_f[2].split("-")
        min = 1 if len(split_f) < 4 else int(split_f[3])
        max = min + 1 if len(split_f) < 5 else int(split_f[4])
        count = generate_count(min, max)
        words = []
        for _ in range(count):
            words.append(""+random.choice(text))
        return_val = " ".join(words)

    return field_name, return_val


def generate_count(min, max):
    if min == max:
        return max
    elif min > max:
        return random.randrange(max, min);
    else:
        return random.randrange(min, max);


def generate_random_doc(format):
    global id_counter

    res = {}

    for f in format:
        f_key, f_val = get_data_for_format(f)
        if f_key:
            res[f_key] = f_val

    if not tornado.options.options.id_type:
        return res

    if tornado.options.options.id_type == 'int':
        res['_id'] = id_counter
        id_counter += 1
    elif tornado.options.options.id_type == 'uuid4':
        res['_id'] = str(uuid.uuid4())

    return res


def set_index_refresh(val):

    params = {"index": {"refresh_interval": val}}
    body = json.dumps(params)
    url = "%s/%s/_settings" % (tornado.options.options.es_url, tornado.options.options.index_name)
    try:
        request = tornado.httpclient.HTTPRequest(url, headers=headers, method="PUT", body=body, request_timeout=240, auth_username=tornado.options.options.username, auth_password=tornado.options.options.password, validate_cert=tornado.options.options.validate_cert)
        http_client = tornado.httpclient.HTTPClient()
        http_client.fetch(request)
        logging.info('Set index refresh to %s' % val)
    except Exception as ex:
        logging.exception(ex)


@tornado.gen.coroutine
def generate_test_data():

    global upload_data_count

    if tornado.options.options.force_init_index:
        delete_index(tornado.options.options.index_name)

    create_index(tornado.options.options.index_name)

    # todo: query what refresh is set to, then restore later
    if tornado.options.options.set_refresh:
        set_index_refresh("-1")

    if tornado.options.options.out_file:
        out_file = open(tornado.options.options.out_file, "w")
    else:
        out_file = None

    if tornado.options.options.dict_file:
        global _dict_data
        with open(tornado.options.options.dict_file, 'r') as f:
            _dict_data = f.readlines()
        logging.info("Loaded %d words from the %s" % (len(_dict_data), tornado.options.options.dict_file))

    format = tornado.options.options.format.split(',')
    if not format:
        logging.error('invalid format')
        exit(1)

    ts_start = int(time.time())
    upload_data_txt = ""
    total_uploaded = 0

    logging.info("Generating %d docs, upload batch size is %d" % (tornado.options.options.count,
                                                                  tornado.options.options.batch_size))
    for num in range(0, tornado.options.options.count):

        item = generate_random_doc(format)

        if out_file:
            out_file.write("%s\n" % json.dumps(item))

        cmd = {'index': {'_index': tornado.options.options.index_name,
                         '_type': tornado.options.options.index_type}}
        if '_id' in item:
            cmd['index']['_id'] = item['_id']

        upload_data_txt += json.dumps(cmd) + "\n"
        upload_data_txt += json.dumps(item) + "\n"
        upload_data_count += 1

        if upload_data_count % tornado.options.options.batch_size == 0:
            yield upload_batch(upload_data_txt)
            upload_data_txt = ""

    # upload remaining items in `upload_data_txt`
    if upload_data_txt:
        yield upload_batch(upload_data_txt)

    if tornado.options.options.set_refresh:
        set_index_refresh("1s")

    if out_file:
        out_file.close()

    took_secs = int(time.time() - ts_start)

    logging.info("Done - total docs uploaded: %d, took %d seconds" % (tornado.options.options.count, took_secs))


if __name__ == '__main__':
    tornado.options.define("es_url", type=str, default='http://localhost:9200/', help="URL of your Elasticsearch node")
    tornado.options.define("index_name", type=str, default='test_data', help="Name of the index to store your messages")
    tornado.options.define("index_type", type=str, default='test_type', help="Type")
    tornado.options.define("batch_size", type=int, default=1000, help="Elasticsearch bulk index batch size")
    tornado.options.define("num_of_shards", type=int, default=2, help="Number of shards for ES index")
    tornado.options.define("http_upload_timeout", type=int, default=3, help="Timeout in seconds when uploading data")
    tornado.options.define("count", type=int, default=100000, help="Number of docs to generate")
    tornado.options.define("format", type=str, default='name:str,age:int,last_updated:ts', help="message format")
    tornado.options.define("num_of_replicas", type=int, default=0, help="Number of replicas for ES index")
    tornado.options.define("force_init_index", type=bool, default=False, help="Force deleting and re-initializing the Elasticsearch index")
    tornado.options.define("set_refresh", type=bool, default=False, help="Set refresh rate to -1 before starting the upload")
    tornado.options.define("out_file", type=str, default=False, help="If set, write test data to out_file as well.")
    tornado.options.define("id_type", type=str, default=None, help="Type of 'id' to use for the docs, valid settings are int and uuid4, None is default")
    tornado.options.define("dict_file", type=str, default=None, help="Name of dictionary file to use")
    tornado.options.define("username", type=str, default=None, help="Username for elasticsearch")
    tornado.options.define("password", type=str, default=None, help="Password for elasticsearch")
    tornado.options.define("validate_cert", type=bool, default=True, help="SSL validate_cert for requests. Use false for self-signed certificates.")
    tornado.options.parse_command_line()

    tornado.ioloop.IOLoop.instance().run_sync(generate_test_data)
   ```
   <br/> To execute the script use following commands:
   ```
   pip install numpy tornado
   nohup python es_test_data.py --es_url=http://localhost:9200 --count=50000000
   ```
   
   ## References:
   https://github.com/oliver006/elasticsearch-test-data <br/>
   https://logz.io/blog/elasticsearch-cluster-tutorial/
   
