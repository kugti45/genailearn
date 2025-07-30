
```python

import csv
import os
from google.cloud import compute_v1

def get_projects_list():
    """Reads projects from projectsinv file in script directory."""
    script_dir = os.path.dirname(os.path.abspath(__file__))
    projects_file = os.path.join(script_dir, 'projectsinv')
    
    with open(projects_file, 'r') as f:
        projects = [line.strip() for line in f if line.strip()]
    return projects

def find_unattached_disks(project_id):
    """Finds all unattached disks in a GCP project."""
    disk_client = compute_v1.DisksClient()
    zone_operations_client = compute_v1.ZoneOperationsClient()
    
    unattached_disks = []
    
    # List all zones in the project
    zones_client = compute_v1.ZonesClient()
    zones = zones_client.list(project=project_id)
    
    for zone in zones:
        zone_name = zone.name
        
        # List all disks in the zone
        disks = disk_client.list(project=project_id, zone=zone_name)
        
        for disk in disks:
            if not disk.users:
                unattached_disks.append({
                    'Project': project_id,
                    'Disk Name': disk.name,
                    'Zone': zone_name,
                    'Size (GB)': disk.size_gb,
                    'Type': disk.type_.split('/')[-1],  # Extract disk type
                    'Status': disk.status,
                    'Creation Timestamp': disk.creation_timestamp,
                    'Last Attach Timestamp': disk.last_attach_timestamp,
                    'Last Detach Timestamp': disk.last_detach_timestamp
                })
    
    return unattached_disks

def export_to_csv(unattached_disks, filename="unattached_disks.csv"):
    """Exports unattached disks data to CSV."""
    if not unattached_disks:
        print("No unattached disks found to export.")
        return
    
    fieldnames = [
        'Project',
        'Disk Name',
        'Zone',
        'Size (GB)',
        'Type',
        'Status',
        'Creation Timestamp',
        'Last Attach Timestamp',
        'Last Detach Timestamp'
    ]
    
    with open(filename, mode='w', newline='') as csvfile:
        writer = csv.DictWriter(csvfile, fieldnames=fieldnames)
        writer.writeheader()
        writer.writerows(unattached_disks)
    
    print(f"Data exported to {filename}")

def main():
    """Main function to find unattached disks across all projects."""
    projects = get_projects_list()
    all_unattached_disks = []
    
    for project in projects:
        print(f"Checking project: {project}")
        try:
            unattached_disks = find_unattached_disks(project)
            all_unattached_disks.extend(unattached_disks)
            print(f"Found {len(unattached_disks)} unattached disks in {project}")
        except Exception as e:
            print(f"Error processing project {project}: {str(e)}")
            continue
    
    if all_unattached_disks:
        print(f"\nTotal unattached disks found: {len(all_unattached_disks)}")
        export_to_csv(all_unattached_disks)
    else:
        print("No unattached disks found in any project.")

if __name__ == "__main__":
    main()

```
```python

import csv
import os
from datetime import datetime, timedelta
from google.cloud import compute_v1
from google.cloud import monitoring_v3

def read_project_list(filename="projectsinv"):
    """Reads project IDs from a file in the same directory."""
    project_list = []
    try:
        with open(filename, 'r') as f:
            for line in f:
                line = line.strip()
                if line and not line.startswith('#'):  # Skip empty lines and comments
                    project_list.append(line)
        return project_list
    except FileNotFoundError:
        print(f"Error: Project list file '{filename}' not found in the current directory.")
        return []
    except Exception as e:
        print(f"Error reading project list: {str(e)}")
        return []

def list_all_vms_with_cpu_metrics(project_id):
    """Lists all VMs with CPU metrics for the past 14 days in a single project."""
    try:
        compute_client = compute_v1.InstancesClient()
        monitoring_client = monitoring_v3.MetricServiceClient()
        
        # Get all VM instances
        request = compute_v1.AggregatedListInstancesRequest(project=project_id)
        aggregated_list = compute_client.aggregated_list(request=request)

        # Prepare time range (last 14 days including today)
        end_time = datetime.utcnow()
        start_time = end_time - timedelta(days=13)  # 14 days total (including today)
        
        project_vms = []
        
        for zone, scoped_list in aggregated_list:
            if scoped_list.instances:
                for instance in scoped_list.instances:
                    print(f"Processing VM: {instance.name} in project {project_id}")
                    
                    # Extract zone and region
                    zone_full = zone.split('/')[-1]
                    region = '-'.join(zone_full.split('-')[:-1])
                    
                    # Get VM creation date
                    creation_date = datetime.strptime(instance.creation_timestamp, "%Y-%m-%dT%H:%M:%S.%f%z").replace(tzinfo=None)
                    
                    # Calculate how many days the VM has existed (up to 14)
                    days_existed = min(14, (end_time - creation_date).days + 1)  # +1 to include today
                    
                    # Get CPU metrics
                    cpu_metrics = get_cpu_metrics(
                        monitoring_client,
                        project_id,
                        instance.name,
                        zone_full,
                        start_time,
                        end_time
                    )
                    
                    # Format metrics with the exact column names
                    formatted_metrics = {}
                    for days_ago in range(13, -1, -1):  # From 13 days ago to today (0 days ago)
                        date = (end_time - timedelta(days=days_ago)).strftime('%Y-%m-%d')
                        
                        # Only include metrics for days the VM existed
                        if days_ago < days_existed:
                            metric_value = cpu_metrics.get(date, "N/A")
                            if days_ago == 0:
                                column_name = "cpuutiliztion today"
                            else:
                                column_name = f"cpuutiliztion {days_ago}days gao"
                            formatted_metrics[column_name] = metric_value
                        else:
                            if days_ago == 0:
                                column_name = "cpuutiliztion today"
                            else:
                                column_name = f"cpuutiliztion {days_ago}days gao"
                            formatted_metrics[column_name] = "N/A"
                    
                    project_vms.append({
                        "Project ID": project_id,
                        "VM Name": instance.name,
                        "Machine Type": instance.machine_type.split('/')[-1] if instance.machine_type else "N/A",
                        "Region": region,
                        "Zone": zone_full,
                        "Status": instance.status,
                        **formatted_metrics
                    })
        
        return project_vms
    
    except Exception as e:
        print(f"Error processing project {project_id}: {str(e)}")
        return []

def get_cpu_metrics(client, project_id, instance_name, zone, start_time, end_time):
    """Fetches max CPU utilization metrics for a VM."""
    project_name = f"projects/{project_id}"
    
    # Prepare the time interval
    interval = monitoring_v3.TimeInterval({
        "end_time": {"seconds": int(end_time.timestamp())},
        "start_time": {"seconds": int(start_time.timestamp())}
    })
    
    # Prepare the metric filter
    filter_str = (
        f'metric.type = "compute.googleapis.com/instance/cpu/utilization" '
        f'AND resource.labels.instance_id = "{instance_name}" '
        f'AND resource.labels.zone = "{zone}"'
    )
    
    # Create the request
    request = monitoring_v3.ListTimeSeriesRequest({
        "name": project_name,
        "filter": filter_str,
        "interval": interval,
        "view": monitoring_v3.ListTimeSeriesRequest.TimeSeriesView.FULL,
        "aggregation": {
            "alignment_period": {"seconds": 86400},  # 1 day
            "per_series_aligner": monitoring_v3.Aggregation.Aligner.ALIGN_MAX,
            "cross_series_reducer": monitoring_v3.Aggregation.Reducer.REDUCE_MAX
        }
    })
    
    # Fetch the metrics
    results = client.list_time_series(request)
    
    # Process the results into daily max values
    cpu_metrics = {}
    for series in results:
        for point in series.points:
            date = datetime.fromtimestamp(point.interval.end_time.seconds).strftime('%Y-%m-%d')
            cpu_metrics[date] = round(point.value.double_value * 100, 2)  # Convert to percentage and round
    
    return cpu_metrics

def export_to_csv(all_vms, filename="vm_cpu_metrics.csv"):
    """Exports VM data with CPU metrics to CSV."""
    if not all_vms:
        print("No VMs found to export.")
        return
    
    # Get all fieldnames from the first VM (they should all have the same structure)
    fieldnames = list(all_vms[0].keys())
    
    with open(filename, mode='w', newline='') as csvfile:
        writer = csv.DictWriter(csvfile, fieldnames=fieldnames)
        writer.writeheader()
        writer.writerows(all_vms)
    
    print(f"Data exported to {filename}")

if __name__ == "__main__":
    # Read project list from file
    project_list = read_project_list()
    
    if not project_list:
        print("No projects to analyze. Please check your projectsinv file.")
        exit()
    
    print(f"Starting VM CPU analysis for {len(project_list)} projects...")
    
    all_vms = []
    for project_id in project_list:
        project_vms = list_all_vms_with_cpu_metrics(project_id)
        all_vms.extend(project_vms)
    
    if all_vms:
        print(f"\nFound {len(all_vms)} VMs across {len(project_list)} projects")
        export_to_csv(all_vms)
        
        # Print summary
        projects_with_vms = len({vm['Project ID'] for vm in all_vms})
        print(f"\nSummary:")
        print(f"Projects analyzed: {len(project_list)}")
        print(f"Projects with VMs: {projects_with_vms}")
        print(f"Total VMs found: {len(all_vms)}")
    else:
        print("No VMs found in any of the projects")
```
```python

import csv
import os
from google.cloud import compute_v1
from google.auth import default
import warnings

# Suppress quota project warning
warnings.filterwarnings("ignore", "Your application has authenticated")

def get_projects_list():
    """Read project IDs from projectsinv file"""
    script_dir = os.path.dirname(os.path.abspath(__file__))
    projects_file = os.path.join(script_dir, 'projectsinv')
    
    try:
        with open(projects_file, 'r') as f:
            return [line.strip() for line in f if line.strip() and not line.startswith('#')]
    except Exception as e:
        print(f"Error reading projectsinv: {str(e)}")
        return []

def list_vms(project_id, credentials):
    """List all VMs in a project"""
    instances_client = compute_v1.InstancesClient(credentials=credentials)
    zones_client = compute_v1.ZonesClient(credentials=credentials)
    
    vms = []
    try:
        for zone in zones_client.list(project=project_id):
            instances = instances_client.list(project=project_id, zone=zone.name)
            for instance in instances:
                vms.append({
                    'Project': project_id,
                    'Name': instance.name,
                    'Zone': zone.name,
                    'Status': instance.status,
                    'MachineType': instance.machine_type.split('/')[-1],
                    'InternalIP': instance.network_interfaces[0].network_i_p if instance.network_interfaces else None,
                    'ExternalIP': instance.network_interfaces[0].access_configs[0].nat_i_p if instance.network_interfaces and instance.network_interfaces[0].access_configs else None
                })
    except Exception as e:
        print(f"Error in {project_id}: {str(e)}")
    return vms

def main():
    projects = get_projects_list()
    if not projects:
        print("No projects found in projectsinv file")
        return
    
    # Use gcloud credentials
    credentials, _ = default()
    
    all_vms = []
    for project in projects:
        print(f"Processing {project}...")
        project_vms = list_vms(project, credentials)
        all_vms.extend(project_vms)
        print(f"Found {len(project_vms)} VMs")
    
    if all_vms:
        with open('vms_report.csv', 'w', newline='') as f:
            writer = csv.DictWriter(f, fieldnames=all_vms[0].keys())
            writer.writeheader()
            writer.writerows(all_vms)
        print(f"\nReport saved to vms_report.csv ({len(all_vms)} VMs total)")
    else:
        print("\nNo VMs found in any project")

if __name__ == "__main__":
    main()

   ``` 
