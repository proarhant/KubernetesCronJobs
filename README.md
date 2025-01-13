# Kubernetes CronJob Scheduling, Configurations and Considerations
Prior migrating OS cron jobs from the server environmen to Kubernetes, we recommend the importance of through testing of the cronjobs on non-prod K8s environment before deploying them into production. The non-production phase will help us identify and resolve any problems before they cause issues when running them on the production.

The Kubernetes Document [https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/] states following:

```
CronJobs have limitations and idiosyncrasies. For example, in certain circumstances, a single CronJob can create multiple concurrent Jobs.
```

Using Helm templates, we should schedule a number of jobs that progressively grows from several to many for instance 10, with closely simulating production workload. We will then properly do the capacity planning for our K8s clusters, what resolutions (e.g. CronJob specs, Pod resource limits, etc) we have to implement to run the production CronJobs.

The considerations should include but not limited to:

1. Appropriate specs (including backoffLimit , restartPolicy, startingDeadlineSeconds, concurrencyPolicy ) for the CronJob resources

2. The idempotent nature of commands used in the jobs

# Scenario 1: 
The following example shows an unwanted behaviour from the specs not suitable even for a very simple shell script that merely calls curl and then checks diff between two files:

The content of values.yaml have following values that are the root cause of the issue e.g. Over 10 Pods (with many restarts) are running within several minutes wheres as only 1 single pod is expected from the cronjob at any given time.

```
 restartPolicy: OnFailure
 resources:
   limits:
```
# Solution to the Scenario 1:
The appropriate specs solve the underlying issue.
```
restartPolicy: Never
concurrencyPolicy: Forbid
startingDeadlineSeconds: 7
successfulJobsHistoryLimit: 3
failedJobsHistoryLimit: 2
     resources:
       limits:
         cpu: "250m"
         memory: "256Mi"
       requests:
         cpu: "125m"
         memory: "128Mi"
```

The shell script used in this CronJob:

```
>cat templates/cronjob-check-ips-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: "{{ .Chart.Name }}"
  namespace: {{ .Values.namespace }}
data:
  ips-current.ini: |
    {
      "Washington, DC, USA" : [ "44.202.178.0/24", "44.202.180.0/23", "44.210.68.0/24", "44.210.110.0/25" ],
      "Columbus, OH, USA" : [ "3.145.224.0/24", "3.145.225.0/25", "3.145.234.0/24" ],
      "San Francisco, CA, USA" : [ "3.101.204.0/23", "3.101.212.0/24", "3.101.209.192/26" ],
      "Portland, OR, USA" : [ "35.89.46.0/23", "35.92.27.0/24" ],
      "Montreal, Québec, CA" : [ "3.99.200.0/24", "3.99.193.0/26", "3.99.253.128/25" ],
      "Dublin, IE" : [ "3.251.231.0/24", "3.251.230.64/26", "3.252.47.0/25" ],
      "London, England, UK" : [ "13.40.201.0/24", "13.40.208.0/25", "13.41.206.128/25", "13.41.206.64/26" ],
      "Paris, FR" : [ "13.38.68.128/25", "13.38.68.64/26", "13.38.202.128/26", "13.38.202.192/28" ],
      "Frankfurt, DE" : [ "3.71.170.0/24", "3.71.103.96/27", "3.75.4.128/25" ],
      "Stockholm, SE" : [ "16.16.1.128/25", "13.50.68.0/26" ],
      "Milan, IT" : [ "15.160.105.128/25", "18.102.58.0/26" ],
      "Tokyo, JP" : [ "35.77.208.0/24", "35.79.233.64/26", "35.79.233.128/28" ],
      "Seoul, KR" : [ "3.38.229.128/25", "15.165.193.192/26" ],
      "Singapore, SG" : [ "13.214.242.0/25", "13.214.223.192/26", "18.141.238.128/25" ],
      "Sydney, AU" : [ "3.26.252.0/24", "3.26.245.128/25", "3.27.51.0/25" ],
      "Mumbai, IN" : [ "3.110.73.192/26", "3.111.138.0/25", "43.205.150.128/26", "43.204.166.240/28" ],
      "Hong Kong, HK" : [ "16.163.86.128/26", "16.163.220.0/25", "43.198.20.64/26", "43.198.20.128/28" ],
      "São Paulo, BR" : [ "15.228.171.128/25", "15.229.44.0/26", "15.229.52.128/26", "15.229.52.80/28" ],
      "Manama, BH" : [ "157.241.17.0/26", "157.241.55.64/26" ],
      "Cape Town, ZA" : [ "13.245.248.192/26", "13.245.248.160/27", "13.246.147.0/26" ]
    }
  cronjob-check-ips.sh: |
                    apt update && apt install -y curl
                    sleep 3
                    cd /cronJobs
                    curl https://s3.amazonaws.com/ip-dnsname/production/ip-ranges.json > /cronJobs/job-01-output/IP_LIST_NOW.txt
                    echo >> /cronJobs/job-01-output/IP_LIST_NOW.txt
                    cp /cronJobs/nr-ips-current.ini /cronJobs/job-01-output/IP_LIST_PROD.txt
                    cd /cronJobs/job-01-output/
                    sed -i 's/[ \t]*/' /cronJobs/job-01-output/IP_LIST_NOW.txt
                    [[ `diff <(sort IP_LIST_PROD.txt) (sort IP_LIST_NOW.txt)` ]] && (curl -X POST -H 'Content-type : application/json' --data '{"text" "K8sCron-IP-RANGES: YES, Change in the Public Service IP list."}' https://hooks.slack.com/services/rrr/xxx) || (curl -X POST -H 'Content-type: application/json' --data '{"text" "K8sCron-IP-RANGES: NO Change in the NR IP list."}' https://hooks.slack.com/services/rrr/xxx)
  cronjob-check-pod-ips.sh: |
                   echo "K8sCron-CHECK-POD-IPS: Are we soon running out of IPs?"
 

```
