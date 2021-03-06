# Filesystem Sync

The default NFS filesystems that Outrigger sets up provides easy sharing of code and files with your project containers. 
However, NFS can be slow when writing, reading, or scanning thousands of files when compared to native filesystem performance. 
Some modern tool kits and package managers favor large numbers of small files and libraries so maintaining high performance 
of a build becomes challenging when your code base is on a volume mounted NFS share.

NFS also has the downside of not propagating filesystem modification events. This can be problematic if you want to run
a process in your container to rebuild or update compiled project assets when modification notifications are triggered
due to local file editing in your IDE.

In order to better support these types of operations, Outrigger provides support for multi-directional filesystem syncing 
(which includes filesystem notifications) using `unison`.  Your project root folder is synced via `unison` into a sync 
container and exposed as a Docker Volume to the rest of your application. This volume is then used as your mount point 
instead of the NFS share and allows nearly native filesystem performance and features.

## Sync Volume Name

The name of the sync volume can be set in a variety of ways. It is determined using the following precedence:

1. Argument provided to the `project sync:start` command
2. Specified in the `sync` -> `volume-name` property of your [Outrigger Project Configuration](./project-configuration.md)
3. In your `docker-compose.yml` file, specified as an `external` volume with a name of the pattern `*-sync`
4. If not specified anywhere else, it will use the name of the current project folder with `-sync` appended

We recommend using approach #3, as it is the most explicit and is in line with all of the configuration you will need
to take full advantage of `unison` syncing.

## Setting up sync for your containers

To setup `unison` sync for your containers you will need to [reference an external volume](https://docs.docker.com/compose/compose-file/#volume-configuration-reference) 
in your `docker-compose.yml` and use that external volume in the mount specification for any services that need to 
reference the same set of files to ensure all containers stay in sync. Also specifying the sync volume in your `build.yml`
file for any containers that reference your code base will have a considerable performance improvement.

**Add the volume spec to your compose file**

```yaml
volumes:
  project-sync:
    external: true
```

!!! important "Note about volume name"
    If you optionally specified a volume name to the `project sync:start` command or configured a name in your Outrigger 
    Project Configuration file the name in this volumes section must match the one specified to the `project sync:start`
    command. The reason why we don't exclusively grab the volume name from the compose file is that this volume may be 
    managed outside of Outrigger by another tool and we want to support named volumes that don't precisely follow 
    the `*-sync` convention we lay out here. 

**Use the external volume as your local mount point**

```yaml
services:
  build:
    image: outrigger/build:php71
    volumes:
      - project-sync:/var/www
```

## Start it up

From your project root run the command `rig project sync:start` (there is also an alias `rig project sync`) to get going.  
With the above configuration you should be up and running. The initial sync can take a few seconds depending on the size
of your project folder. You will see a progress indicator that will let you know when the initial sync is finished and 
things are ready to use.


## Cleaning up

Since the `project sync:start` command starts another container to manage the volume syncing, we have a command `project sync:stop`
that will clean up that running container for you. It discovers the volume / container name the same way `project sync:start` 
does.

When you are done for the day, or for that project, from the project root `rig project sync:stop` will clean up any running
`unison` containers for that project.
