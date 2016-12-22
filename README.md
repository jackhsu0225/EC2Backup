 public class Program
    {
        public static IAmazonEC2 Ec2Client;
        public static void Main(string[] args)
        {
            Ec2Client = AWSClientFactory.CreateAmazonEC2Client();
            
            // Get a list of all of your Ec2 volumes
            var describeVolumesResponse = Ec2Client.DescribeVolumes();
            var volumeIds = describeVolumesResponse.Volumes.Select(vr => vr.VolumeId).ToList();
            // Delete any Snapshots older than 10 days
            DeleteOldSnapshots(volumeIds, 10);
            // Create a new Snapshot for each volume
            CreateNewSnapshots(volumeIds);            
            
            Console.Read();
        }
        public static void DeleteOldSnapshots(List<string> volumeIds, int maxDays)
        {
            var describeSnapshotsRequest = new DescribeSnapshotsRequest
            {
                Filters = new List<Filter> { new Filter { Name = "volume-id", Values = volumeIds } }
            };
            var describeSnapshotsResponse = Ec2Client.DescribeSnapshots(describeSnapshotsRequest);
            foreach (var snapshot in describeSnapshotsResponse.Snapshots)
            {
                var age = (DateTime.UtcNow - snapshot.StartTime.ToUniversalTime()).TotalDays;
                Console.WriteLine("Description: {0} Age:{1}", snapshot.Description, age);
                if (age > maxDays)
                {
                    Console.WriteLine("Deleting ");
                    Ec2Client.DeleteSnapshot(new DeleteSnapshotRequest {SnapshotId = snapshot.SnapshotId});
                }
            }
        }
        public static void CreateNewSnapshots(List<string> volumeIds)
        {
            foreach (var volume in volumeIds)
            {
                var description = string.Format("{0} vol={1}", DateTime.UtcNow.ToShortDateString(), volume);
                var createSnapshotRequest = new CreateSnapshotRequest {Description = description, VolumeId = volume};
                var response = Ec2Client.CreateSnapshot(createSnapshotRequest);
                Console.WriteLine("Snapshot:{0} of Volume:{1} created", response.Snapshot.SnapshotId, volume);
            }
        }

    }
}
