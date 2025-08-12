
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
24 hours metrics

```
from googleapiclient import discovery
from google.cloud import monitoring_v3
from datetime import datetime, timedelta
import pytz
import pandas as pd

PROJECT_ID = "weighty-bounty-467017-r3"

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

def get_instance_id(project, zone, name):
    service = discovery.build('compute', 'v1')
    request = service.instances().get(project=project, zone=zone, instance=name)
    response = request.execute()
    return response['id']

def get_max_cpu_utilization(project_id, instance_name, zone, start_time, end_time):
    client = monitoring_v3.MetricServiceClient()
    instance_id = get_instance_id(project_id, zone, instance_name)

    instance_filter = (
        f'metric.type="compute.googleapis.com/instance/cpu/utilization" '
        f'AND resource.labels.instance_id="{instance_id}"'
    )

    interval = monitoring_v3.TimeInterval({
        "start_time": start_time,
        "end_time": end_time,
    })

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

if __name__ == "__main__":
    instances = list_running_instances(PROJECT_ID)
    print(f"Found {len(instances)} running VM(s) in project {PROJECT_ID}:\n")

    vm_data = []
    now = datetime.utcnow().replace(tzinfo=pytz.UTC)

    for instance in instances:
        print(f"=== VM: {instance['name']} (Zone: {instance['zone']}) ===")
        instance_id = get_instance_id(PROJECT_ID, instance['zone'], instance['name'])

        row_data = {
            "VM Name": instance['name'],
            "Zone": instance['zone'],
            "Instance ID": instance_id
        }

        # Collect hourly utilization for past 24 hours
        for i in range(23, -1, -1):
            start_time = now - timedelta(hours=i+1)
            end_time = now - timedelta(hours=i)
            try:
                utilization = get_max_cpu_utilization(
                    PROJECT_ID, instance['name'], instance['zone'], start_time, end_time
                )
            except Exception as e:
                utilization = None

            if i == 0:
                row_data["current"] = utilization
                print(f"current max cpu utilization: {utilization}%")
            else:
                row_data[f"{i}h_ago"] = utilization
                print(f"{i} hours ago max cpu utilization: {utilization}%")

        vm_data.append(row_data)

    # Create DataFrame and save to Excel
    df = pd.DataFrame(vm_data)
    output_file = "gcp_vm_cpu_metrics.xlsx"
    df.to_excel(output_file, index=False)
    print(f"\nMetrics saved to {output_file}")
```
24 days metrics
```
from googleapiclient import discovery
from google.cloud import monitoring_v3
from datetime import datetime, timedelta
import pytz
import pandas as pd

PROJECT_ID = "weighty-bounty-467017-r3"

def list_running_instances(project):
    service = discovery.build('compute', 'v1')
    request = service.instances().aggregatedList(project=project)
    instances_info = []

    while request is not None:
        response = request.execute()
        for zone, zone_items in response.get('items', {}).items():
            for instance in zone_items.get('instances', []):
                creation_time_str = instance['creationTimestamp']
                creation_time = datetime.strptime(creation_time_str, "%Y-%m-%dT%H:%M:%S.%f%z")

                instances_info.append({
                    'name': instance['name'],
                    'zone': instance['zone'].split('/')[-1],
                    'creation_date': creation_time
                })
        request = service.instances().aggregatedList_next(previous_request=request, previous_response=response)

    return instances_info

def get_instance_id(project, zone, name):
    service = discovery.build('compute', 'v1')
    request = service.instances().get(project=project, zone=zone, instance=name)
    response = request.execute()
    return response['id']

def get_max_cpu_utilization(project_id, instance_name, zone, start_time, end_time):
    client = monitoring_v3.MetricServiceClient()
    instance_id = get_instance_id(project_id, zone, instance_name)

    instance_filter = (
        f'metric.type="compute.googleapis.com/instance/cpu/utilization" '
        f'AND resource.labels.instance_id="{instance_id}"'
    )

    interval = monitoring_v3.TimeInterval({
        "start_time": start_time,
        "end_time": end_time,
    })

    results = list(client.list_time_series(
        request={
            "name": f"projects/{project_id}",
            "filter": instance_filter,
            "interval": interval,
            "view": monitoring_v3.ListTimeSeriesRequest.TimeSeriesView.FULL,
        }
    ))

    if not results:  # No data available for this time period
        return None

    max_utilization = 0.0
    for result in results:
        for point in result.points:
            value = point.value.double_value
            if value > max_utilization:
                max_utilization = value

    return round(max_utilization * 100, 2)

if __name__ == "__main__":
    instances = list_running_instances(PROJECT_ID)
    print(f"Found {len(instances)} VM(s) in project {PROJECT_ID}:\n")

    vm_data = []
    now = datetime.utcnow().replace(tzinfo=pytz.UTC)

    for instance in instances:
        print(f"=== VM: {instance['name']} (Zone: {instance['zone']}) ===")
        instance_id = get_instance_id(PROJECT_ID, instance['zone'], instance['name'])
        creation_date = instance['creation_date']

        row_data = {
            "VM Name": instance['name'],
            "Zone": instance['zone'],
            "Instance ID": instance_id,
            "Creation Date": creation_date.strftime("%Y-%m-%d %H:%M:%S %Z")
        }

        for i in range(24, 0, -1):
            start_time = now - timedelta(days=i)
            end_time = start_time + timedelta(days=1)

            # Check if VM existed at that day
            if start_time < creation_date:
                utilization = "N/A"
            else:
                utilization = get_max_cpu_utilization(
                    PROJECT_ID, instance['name'], instance['zone'], start_time, end_time
                )
                if utilization is None:
                    utilization = "N/A"

            row_data[f"{i}d_ago"] = utilization
            print(f"{i} days ago max cpu utilization: {utilization}")

        vm_data.append(row_data)

    # Save results to Excel
    df = pd.DataFrame(vm_data)
    output_file = "gcp_vm_cpu_metrics_24days_with_creation.xlsx"
    df.to_excel(output_file, index=False)
    print(f"\nMetrics saved to {output_file}")

```
