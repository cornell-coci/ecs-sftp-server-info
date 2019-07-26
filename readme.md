# SFTP on ECS

## goals:
 - run sftp with HA on ecs cluster
 - have security groups to control access
 - work over public internet or direct connect from campus network

## Tech used:
 - docker for sftp container; atmoz/sftp - forked to our own repo.
 - route 53 service discovery
 - lamba
 - cloudwatch events
 - ecs
 - efs
 - ELB - NLB class

## how it works:

### overview:
The atmoz sftp image is small and fast, packaging openssh and a few wrapper shell scripts to create users.  Bind mounting through docker enables persistence.  To make this Highly Available, the container is put behind a Network Load Balancer.  The NLB listens on port a static, high port and forwards to the container port.  The desired count is one, which is enough to support all of the reasons we use it.

### file systems:
File system is simple.  The sftp container runs as root but not privileged.  It is configured through environment variables and public keys in it's mounted directory structure.  To keep it fast and because keys change infrequently, keys are written by an external process.  Keys are stored in an individual file, then combined into authorized_keys when the container starts.  That means that adding or removing keys requires a container restart.  The home directory is bind mounted into the container.  On the host, this is an EFS/NFS volume.  The container does not care.

### users:
We use ansible, so our task definition is a Jinja2 template.  The service's yaml config has a list of user names, which correspond to a second list of users with groups and IDs.  When run, this configures an environmental variable in the task defintion according to the documentation.  Keys are mounted into the containers file system for each user.

### networking:
Container networking gets interesting.  AWSVPC network mode would make sense except that the NAT translation replaces the source IP with the NLB ip.  This makes security groups useless.  Instead, bridge network mode is used so that we can apply security groups.  ECS by default uses ports above ~32768 for mapping host to container ports.  We set the container port to 30000 to avoid using a port in this range.  That also simplifies the security group rules.  Traffic from the NLB will still have the correct source IP, which will be checked against the security group for the ECS cluster.  The NLB's target group knows where the sftp service is running and routes traffic accordingly.

For traffic in our private network (including Direct Connect from campus), we did not want to use a public network load balancer.  Instead, we can use the private IP of the container instance and the known port of the container.  Since there are many container instances where it could be running, service discovery is a solution to the problem.  


### service discovery:
AWS Service Discovery has a "hosted zone", which is private to your VPC and part of Route53.  When used with ECS, containers that have service discovery enabled will create a "service registry".  This service registry is used to update a SRV record when the associated service state is updated.  Suscintly, containers with service discovery enabled will create DNS records (of type SRV) that we can query to know where services are running.  A SRV record has the format: [priority] [weight] [port] [server host name].

SRV records are useful, but there are some limitations.  First, most programs do not natively understand them.  Second, they only work in your VPC, meaning they don't work over direct connect.  To solve the first problem, we need to translate a SRV record to a port number and a host.  Since the port is always 30000, we only need to figure out the host.  To solve the second problem, we can publish the host IP somewhere that on campus servers can look it up, like an A record on our public DNS.  Using this technique lets us connect to our AWS hosted sftp service over direct connect instead of the public internet.

The implementation is about 115 lines of python that could be cleaned up.  Environmental variables set the internal zone, external zone, internal service record, and external service record.  On execution, the script queries the internal service record, parses out the host, and updates the external A record on the external zone.

To keep the A record up to date, a cloudwatch event rule triggers the lambda function when the service enters the "RUNNING" state.  When a container registers with service discovery, the state is "ACTIVATING" until registered.  After registration, it changes to RUNNING.  Because the transition to RUNNING happens after successful registration, we always get the latest running container address in our A record.
