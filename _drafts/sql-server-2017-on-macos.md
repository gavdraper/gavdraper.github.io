---
layout: post
title: Up and Running With SQL Server 2017 on macOS In Less Than 5 Minutes
date: '2017-06-18 09:35:34'
---
SQL Server 2017 is the first version of SQL Server that will also run natively on Linux and on macOS via Docker. There has neer been a quicker way to get SQL Server installed and running than using this new Docker image.

Let's run through the steps involved, you'll need to have Docker installed to follow along.

1. Run the following command to pull down the SQL Server Docker image

    > docker pull microsoft/mssql-server-linux

1. Now we just need to run this image. Let's also mount a volume so any data we create gets persisted between container restarts this is done by including the -v argument in the Docker run command...

    > Docker run -e 'ACCEPT_EULA=Y' -e 'SA_PASSWORD=(StrongP4ssword)' -p 1433:1433 -v sqlvolume:/var/opt/mssql -d microsoft/mssql-server-linux

Let's break down what this command is doing...

* -e 'ACCET-EULA=Y' : This accepts the EULA and without this SQL Server will not start. This is normally done in the UI For the SQL Server installer but in this case as there is no installer it's done in the Docker Run command.
* -e 'SA_Password = : Here we specify the SA account password that we will use to connect to this server.
* -p 1433:1433 : This publishes port 1433 on the container to port 1433 on the host to allow us to connect to SQL Server.
* -v sqlvolume : This tells our container to mount a volume so we can store files in that path and they will be stored on the host to survive restarts of the container (By default containers store no data and will start in a clean state every time)

At this point we now have a SQL Server instance running in our new container. On macOS the best ways to connect to this are either via VS Code or SqlCmd for a command line option. Whichever tool you choose you just need to point it to localhost with a username of sa and a password of whatever you specified in your Docker Run command.

![Connecting to SQL with SqlCmd]({{site.url}}/content/images/2017-sql-on-macos/sqlcmd.jpg)