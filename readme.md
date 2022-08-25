# Mysql benchmark on kubernetes with sysbench

## Source of inspiration 

- https://github.com/akopytov/sysbench
- https://blog.purestorage.com/purely-informational/how-to-benchmark-mysql-performance/
- https://github.com/Percona-Lab/sysbench-tpcc
- https://www.percona.com/blog/2018/03/05/tpcc-like-workload-sysbench-1-0/
- https://www.tpc.org/tpcc/

## What is sysbench and TPC-C

Sysbench is a very versatile and scalable database performance benchmarking tool. It makes use of .lua files to create test scenarios.

TPC Benchmark C is an on-line transaction processing (OLTP) benchmark standard. TPC-C involves a mix of five concurrent transactions of different types and complexity either executed on-line or queued for deferred execution. The database is comprised of nine types of tables with a wide range of record and population sizes. TPC-C is measured in transactions per minute (tpmC). While the benchmark portrays the activity of a wholesale supplier, TPC-C is not limited to the activity of any particular business segment, but, rather represents any industry that must manage, sell, or distribute a product or service.

We're going to run TPCC-like workload using the percona project [sysbench-tpcc](https://github.com/Percona-Lab/sysbench-tpcc). 


## build the docker image 

We build a docker image that install sysbench and also embed the sysbench-tpcc lua files that will be consumed by sysbench.

``` 
docker build -t sysbench .
# test 
docker run -it sysbench
$> sysbench --version
$> sysbench --help
```

You can also use my repository `michaelcourcy/sysbench`.

## Execute a sysbench test in kubenetes against a mysql database

### Install mysql in your cluster 

```
kubectl create namespace mysql
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  replicas: 1
  template:
    metadata:
      labels:
        app: mysql
    spec:
      securityContext:
        runAsUser: 0
      containers:
      - name: mysql
        image: mysql:8.0.26
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: ultrasecurepassword
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      # storageClassName: basic
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: mysql
  name: mysql
  namespace: mysql
spec:
  ports:
  - name: "3306"
    port: 3306
    protocol: TCP
    targetPort: 3306
  selector:
    app: mysql
  type: ClusterIP
EOF
```

Wait for mysql to be ready.
```
watch kubectl get po -n mysql
```


### Create the sysbench test database sbtest

Create a mysql client to connect to the database
```
kubectl run mysql-client --restart=Never --rm -it --image=mysql:8.0.26 -n mysql -- bash
```
Connect to the server
```
mysql --user=root --password=ultrasecurepassword -h mysql
```
Create database and data
```
CREATE DATABASE sbtest;
show databases;
```

### Prepare the database 

In another terminal launch the sysbench pod.

```
kubectl run sysbench --image=michaelcourcy/sysbench -n mysql --command -- tail -f /dev/null
kubectl exec sysbench -n mysql -it -- bash
```

3 parameters are very important for having an appropriate database size :

* `--threads`	The number of threads to run the operation at. This simulates the number of users addressing the database. Some example values are: 8, 16, 24, 32, 48, 64, 96, 128, 256, 512, and 1024. The higher the thread count the more the system resource usage will be. Default is 1.
* `--tables`	The number of tables to create in the database/schema.
* `--scale`	The scale factor (warehouses) which increases the amount of data to work on overall. Increase –tables and –scale to increase the database size and number of rows operated on. As a rough estimation, 100 warehouses with 1 table set produces about 10GB of data in non-compressed InnoDB tables (so 100 warehouses with 10 table sets gives about 100GB – 50 tables and 500 warehouses offer 2.5TB-3TB of database size.)

Because we are working on a 10Gb database I choose 5 warehouses with 5 tables to have another 5Gb available. 

```
cd sysbench-tpcc
# prepare the database 
sysbench tpcc.lua \
  --threads=10 \
  --tables=5 \
  --scale=5 \
  --report-interval=1 \
  --db-driver=mysql --mysql-host=mysql --mysql-user=root --mysql-password=ultrasecurepassword --mysql-db=sbtest \
  prepare
```

while sysbench finish the preparation of the database you can comeback on the mysql-client terminal and check the content of the database.

```
use sbtest;
show tables;
show columns from warehouse1;
show columns from warehouse2;
show columns from order1;
show columns from district1;
select count("*") from warehouse2;
select count("*") from orders1;
select count("*") from stock6;
# and so on ... 
```

At the end of the preparation in another terminal let's check the size of the disk 
```
kubectl exec -it mysql-0  -- bash 
df | grep /var/lib/mysql
/dev/sdf        10218772  4622876   5579512  46% /var/lib/mysql
du -hs /var/lib/mysql
4.5G    /var/lib/mysql
```

### Execute the benchmark 

In the sysbench pod execute the test 

```
sysbench tpcc.lua \
  --threads=10 \
  --tables=5 \
  --scale=5 \
  --time=600 \
  --report-interval=1 \
  --db-driver=mysql --mysql-host=mysql --mysql-user=root --mysql-password=ultrasecurepassword --mysql-db=sbtest \
  run
```

# Impact of the kasten backup during the execution of the benchmark 

Then bench last 10 minutes 

## Without a kasten backup

```
SQL statistics:
    queries performed:
        read:                            209210
        write:                           216928
        other:                           32278
        total:                           458416
    transactions:                        16124  (26.85 per sec.)
    queries:                             458416 (763.48 per sec.)
    ignored errors:                      76     (0.13 per sec.)
    reconnects:                          0      (0.00 per sec.)

Throughput:
    events/s (eps):                      26.8540
    time elapsed:                        600.4331s
    total number of events:              16124

Latency (ms):
         min:                                    1.22
         avg:                                  372.35
         max:                                 3154.81
         95th percentile:                     1032.01
         sum:                              6003750.30

Threads fairness:
    events (avg/stddev):           1612.4000/33.40
    execution time (avg/stddev):   600.3750/0.04
```


## With a Kasten backup 

```
SQL statistics:
    queries performed:
        read:                            202941
        write:                           210780
        other:                           31242
        total:                           444963
    transactions:                        15606  (25.99 per sec.)
    queries:                             444963 (741.06 per sec.)
    ignored errors:                      76     (0.13 per sec.)
    reconnects:                          0      (0.00 per sec.)

Throughput:
    events/s (eps):                      25.9910
    time elapsed:                        600.4388s
    total number of events:              15606

Latency (ms):
         min:                                    1.36
         avg:                                  384.62
         max:                                 3111.43
         95th percentile:                     1069.86
         sum:                              6002422.45

Threads fairness:
    events (avg/stddev):           1560.6000/54.88
    execution time (avg/stddev):   600.2422/0.11
```

We may notice a slight performance drop on this 10 minutes runs : 
- 3% drop on read 
- 2.8% drop on write 

This is most likely due to the snapshot activity but note that it depends of the storage vendor because Kasten use the capacity of the storage vendor to take snapshot. 


# Restoration

It is interesting to test the restoration of the mysql workload when he was backup under heavy load. 

Delete the mysql namespace and restore it with kasten. We were able to check that  
- server restarted properly 
- database sbtest is here 
- tables are here with their contents 