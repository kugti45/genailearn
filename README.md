Here's a Python code snippet:

```python
def greet(name):
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
