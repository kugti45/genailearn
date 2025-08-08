
```python
from googleapiclient import discovery
from google.cloud import monitoring_v3
from google.oauth2 import service_account
from datetime import datetime, timedelta
import pytz

PROJECT_ID = "weighty-bounty-467017-r3"
ZONE = "-"  # Use "-" to search all zones

def list_running_instances(project):
    service = discovery.build('compute', 'v1')
    request = service.instances().aggregatedList(project=project)
    running_instances = []

    while request is not None:
        response = request.execute()
        for zone, zone_items in response.get('items', {}).items():
            for instance in zone_items.get('instances', []):
                if instance['status'] == 'RUNNING':
                    running_instances.append({
                        'name': instance['name'],
                        'zone': instance['zone'].split('/')[-1]
                    })
        request = service.instances().aggregatedList_next(previous_request=request, previous_response=response)

    return running_instances

def get_max_cpu_utilization(project_id, instance_name, zone):
    client = monitoring_v3.MetricServiceClient()
    now = datetime.utcnow().replace(tzinfo=pytz.UTC)
    interval = monitoring_v3.TimeInterval({
        "end_time": now,
        "start_time": now - timedelta(hours=1),
    })
    instance_filter = (
        f'metric.type="compute.googleapis.com/instance/cpu/utilization" '
        f'AND resource.labels.instance_id="{get_instance_id(project_id, zone, instance_name)}"'
    )

    results = client.list_time_series(
        request={
            "name": f"projects/{project_id}",
            "filter": instance_filter,
            "interval": interval,
            "view": monitoring_v3.ListTimeSeriesRequest.TimeSeriesView.FULL,
        }
    )

    max_utilization = 0.0
    for result in results:
        for point in result.points:
            value = point.value.double_value
            if value > max_utilization:
                max_utilization = value

    return round(max_utilization * 100, 2)

def get_instance_id(project, zone, name):
    service = discovery.build('compute', 'v1')
    request = service.instances().get(project=project, zone=zone, instance=name)
    response = request.execute()
    return response['id']

if __name__ == "__main__":
    instances = list_running_instances(PROJECT_ID)
    print(f"Found {len(instances)} running VM(s) in project {PROJECT_ID}:\n")

    for instance in instances:
        try:
            utilization = get_max_cpu_utilization(PROJECT_ID, instance['name'], instance['zone'])
            print(f"VM: {instance['name']} (Zone: {instance['zone']}) - Max CPU Utilization (last hour): {utilization}%")
        except Exception as e:
            print(f"Error fetching CPU data for {instance['name']}: {e}")

   ```

```
pip install google-api-python-client google-cloud-monitoring
```
