# Database access

Connections to the application database have three levels of login security:

1. The database itself requires a username and password. The API uses a service role, but individual roles are created for users.
2. The database can only be reached through the bastion server. Each user will be issued an account on the bastion server.
3. And the bastion server's security group, `{project}-bastion-sg`, can be used to restrict access by IP address.

## Parameters

- {project}: The project name
- {bastion-username}: An individual user's username on the Bastion server
- {db-host}: The database hostname, determined during application deployment
- {db-username}: An individual user's PostgreSQL database login role name
- {db-password}: An individual user's PostgreSQL database password

## Database connection settings

- **Database type:** PostgreSQL 14
- **Database host:** {db-host}
- **Port:** 5432
- **Database name:** `{project}`
- **Username:** {db-username}
- **Password:** {db-password}
 
## SSH tunnel settings
 
- **Server:** 34.211.167.138
- **Port:** 22
- **Username:** {bastion-username}
- **SSH key:** <bastion SSH key>

## SSH configuration entry for bastion host & SSH tunnels (recommended)

Creating an SSH configuration entry will make access more convenient.

1. Store your bastion key in ~/.ssh/{project}-bastion-{bastion-username}.pem.
2. Change its permissions:
```
chmod 600 ~/.ssh/ {project}-bastion-{bastion-username}.pem
```
3. Add these blocks to ~/.ssh/config:
```
Host {project}-bastion
User {bastion-username}
HostName 34.211.167.138
IdentitiesOnly yes
IdentityFile ~/.ssh/{project}-bastion-{bastion-username}.pem

Host tunnel-{project}-prod-db
HostName 34.211.167.138
User {bastion-username}
IdentitiesOnly yes
IdentityFile ~/.ssh/{project}-bastion-{bastion-username}.pem
LocalForward localhost:5433 {project}-prod-db.crqhdizsvjux.us-west-2.rds.amazonaws.com:5432
```

Now you can open an SSH tunnel to the database:
```
ssh tunnel-{project}-prod-db
```

Leave the tunnel open in its own terminal window. With it running, you can connect to the database using psql, using your database username and password:

```
psql -h 127.0.0.1 -p 5433 <PostgreSQL username>
```

You will be prompted for your PostgreSQL password.

## Connecting from SQL GUI applications
 
Most PostgreSQL GUI software supports connecting through an SSH tunnel and does not require you to create your own tunnel as in the example above.

Some software will use the `{project}-bastion` entry in ~/.ssh/config, while in other cases you may need to enter the connection details and supply the bastion SSH key.