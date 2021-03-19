# Setup

In this file we describe the steps to run the server and web extension of Bitwarden locally.

### Requirements

- [.NET Core 3.1 SDK](https://www.microsoft.com/net/download/core)
- [SQL Server 2017](https://docs.microsoft.com/en-us/sql/index)

_These dependencies are free to use._

### Recommended Development Tooling

- [Visual Studio](https://www.visualstudio.com/vs/) (Windows and macOS)
- [Visual Studio Code](https://code.visualstudio.com/) (other)

_These tools are free to use._

# Server Architecture

The Server is divided into a number of services. Each service is a Visual Studio project in the Server solution. These are:

- Admin
- Api
- Icons
- Identity
- Notifications
- SQL

Each service is built and run separately. The Bitwarden clients can use different servers for different services.

This means that you don't need to run all services locally for a development environment. You can run only those services that you intend to modify, and use Bitwarden.com or a self-hosted instance for all other services required.

# Database

## SQL Server

This guide will show you how to set up the Api, Identity and SQL projects for development. These are the minimum projects for any development work. You may need to set up additional projects depending on the changes you want to make.

We recommend using [Visual Studio](https://visualstudio.microsoft.com/vs/).

### With Docker

We use SQL Server Developer Edition as a Docker Container to run the local development database. More information about the container image is available [on its Docker Hub page](https://hub.docker.com/_/microsoft-mssql-server) (this is especially useful if you're having issues).

Steps:

1. Make sure you have already run `git clone` for the server repo, so that you have the migrator scripts required

2. Install the [Docker desktop runtime](https://hub.docker.com/editions/community/docker-ce-desktop-mac)

3. Create a Docker account if prompted (optional)
4. Organize your folders (important for the below scripts to work properly):
   - Create a folder to house Docker information (e.g. `docker`) - this should be in the same folder as your cloned repositories (e.g. `server`, `web`, `browser` etc)
   - Create sub-folder to house this specific container (ex: `docker/SQLServer`)
5. Create `docker-compose.yml` file in `docker/SQLServer` with the following contents:

   ```Dockerfile
   version: '3.1'
   services:
     db:
       image: mcr.microsoft.com/mssql/server:2017-CU14-ubuntu
       container_name: mssql-dev
       restart: always
       environment:
         ACCEPT_EULA: Y
         SA_PASSWORD: SET_A_PASSWORD_HERE
         MSSQL_PID: Developer
       volumes:
         - mssql_dev_data:/var/opt/mssql/data
         - ../../server/util/Migrator:/mnt/migrator/
       ports:
         - '1433:1433'
   volumes:
     mssql_dev_data:
   ```

6. Update the SA_PASSWORD field with a password you want to use. It must include at least 8 characters of at least three of these four categories: uppercase letters, lowercase letters, numbers and non-alphanumeric symbols.
7. Open up a terminal window and cd into your folder with the .yml file and execute the following command
   - `docker-compose up -d` (omit the `-d` switch if you want to see the output for debugging)

- You should now have a container up and running the SQL Server

<img width="194" alt="salserver" src="https://user-images.githubusercontent.com/3904944/78942922-b344e880-7a88-11ea-8c1e-ba12ab3446bb.png">

### Running Migrator scripts

You now have an empty SQL server instance. The instructions below will automatically create your `vault_dev` database and run the migration scripts located in `server/util/Migrator/DbScripts` to populate it.

Note: you must have followed the steps above to set up your folder structures and the `docker-compose` file for this to work.

1. Open the Docker Desktop GUI
2. Click the CLI button to open a new terminal in your mssql-dev service
   ![Screen Shot 2021-03-18 at 11 12 30 am](https://user-images.githubusercontent.com/31796059/111558643-e59faf80-87da-11eb-96d7-c26875ce322c.png)
3. Run `sh /mnt/migrator/createVaultDev.sh 'SA_PASSWORD'`. Replace `SA_PASSWORD` with the password you set in your `docker-compose.yml` file. You should put your `SA_PASSWORD` in single quotes to avoid special bash characters being caught by the shell.

4. (Optional) To confirm this worked correctly, you can connect to the SQL database with an SQL client (such as [Azure Data Studio](https://docs.microsoft.com/en-us/sql/azure-data-studio/download-azure-data-studio?view=sql-server-ver15)). You should see that the `vault_dev` database has been created and has been populated with tables.

5. **Troubleshooting:** if your login details for `sa` are being rejected:
   - Make sure your SA_PASSWORD is meeting the complexity requirements above. If these requirements are not met, the password may not be set properly (without any warning) and your login attempts will be rejected for having an incorrect password. If this is happening and you're sure you're using the right password, try increasing the complexity of SA_PASSWORD.
   - If you change SA_PASSWORD in `docker-compose.yml`, you may need to delete the Docker container _and volume_ for it to take effect. (This will obviously delete all of your container files/setup.) Stop and delete the running container, then delete the volume with `docker volume ls` and `docker volume rm <volume name>`. Then update `docker-compose.yml` and run `docker compose up -d` again.
   - Make sure you are wrapping your SA_PASSWORD in single quotes when executing the `createVaultDev.sh` script.

Your database is now ready to go!

The SQL server should be running smoothly. :tada:

# Server

## User Secrets

User secrets are a method for managing application settings on a per-developer basis. They are stored outside of the local git repository so that they are not pushed to remote.

User secrets override the settings in `appsettings.json` of each project. Your user secrets file should match the structure of the `appsettings.json` file for the settings you intend to override.

For more information, see: [Safe storage of app secrets in development in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets?view=aspnetcore-3.1).

### Editing user secrets - Visual Studio on Windows

Right-click on the project in the Solution Explorer and click **Manage User Secrets**.

### Editing user secrets - Visual Studio on macOS

Open a terminal and navigate to the project directory, like `src/<project-name>`. For our setup, only `src/Identity` and `src/Api` are relevant.

Once there, initiate and create the blank user secrets file by running:

```bash
dotnet user-secrets init
```

Add a user secret by running:

```bash
dotnet user-secrets set "<key>" "<value>"
```

View currently set secrets by running:

```bash
dotnet user-secrets list
```

## User Secrets - Certificates

Once you have your user secrets files set up, you'll need to generate 3 of your own certificates for use in local development.

This guide uses OpenSSL to generate the certificates. If you are using Windows, pre-compiled OpenSSL binaries are available via [Cygwin](https://www.cygwin.com/).

1. Open a terminal.
2. Create an Identity Server (Dev) certificate file (.crt) and key file (.key):
   ```bash
   openssl req -x509 -newkey rsa:4096 -sha256 -nodes -keyout identity_server_dev.key -out identity_server_dev.crt -subj "/CN=Bitwarden Identity Server Dev" -days 3650
   ```
3. Create an Identity Server (Dev) .pfx file based on the certificate and key you just created. You will be prompted to enter a password - remember this because you’ll need it later:
   ```bash
   openssl pkcs12 -export -out identity_server_dev.pfx -inkey identity_server_dev.key -in identity_server_dev.crt -certfile identity_server_dev.crt
   ```
4. Create a Data Protection (Dev) certificate file (.crt) and key file (.key):
   ```bash
   openssl req -x509 -newkey rsa:4096 -sha256 -nodes -keyout data_protection_dev.key -out data_protection_dev.crt -subj "/CN=Bitwarden Data Protection Dev" -days 3650
   ```
5. Create a Data Protection (Dev) .pfx file based on the certificate and key you just created. You will be prompted to enter a password - remember this because you’ll need it later:
   ```bash
   openssl pkcs12 -export -out data_protection_dev.pfx -inkey data_protection_dev.key -in data_protection_dev.crt -certfile data_protection_dev.crt
   ```
6. Install the .pfx files by double-clicking on them and entering the password when prompted.
   - On Windows, this will add them to your certificate stores. You should add them to the "Trusted Root Certificate Authorities" store.
   - On MacOS, this will add them to your keychain. You should update the Trust options for each certificate to `always trust`.
7. Get the SHA1 thumbprint for the Identity and Data Protection certificates
   - On Windows
     - press Windows key + R to open the Run prompt
     - type "certmgr.msc" and press enter. This will open the system tool used to manage user certificates
     - find the "Bitwarden Data Protection Dev" and "Bitwarden Identity Server Dev" certificates in the Trusted Root Certificate Authorities > Certificates folder
     - double click on the certificate
     - click the "Details" tab and find the "Thumbprint" field in the list of properties.
   - On MacOS
     - press Command + Spacebar to open the Spotlight search
     - type "keychain access" and press enter
     - find the "Bitwarden Data Protection Dev" and "Bitwarden Identity Server Dev" certificates
     - select each certificate and click the "i" (information) button
     - find the SHA-1 fingerprint in the list of properties
8. Add the SHA1 thumbprints of both certificates to your user secrets for the Api and Identity projects. (See the example user secrets file below.)

## User Secrets - Other

**selfhosted**: It is highly recommended that you use the `selfHosted: true` setting when running a local development environment. This tells the system not to use cloud services, assuming that you are running your own local SQL instance.

Alternatively, there are emulators that allow you to run local dev instances of various Azure and/or AWS services (e.g. local-stack), or you can use your own Azure accounts for provisioning the necessary services and set the connection strings accordingly. These are outside the scope of this guide.

**sqlServer\_\_connectionString**: this provides the information required for the Server to connect to the SQL instance. See the example connection string below.

**licenseDirectory**: this must be set to avoid errors, but it can be set to an aribtrary empty folder.

- I'm using macOS and my folder path is: `c:/bwtest/licenses`

**installation\_\_key** and **installation\_\_id**: request your own private Installation Id and Installation Key for self-hosting: https://bitwarden.com/host/.

## Example User Secrets file

This is an example user secrets file for both the Api and Identity projects.

**Note 1:** These user secret files will be found in the folder `~/.microsoft/usersecrets/` for macOS and possibly, for all UNIX systems.

**Note 2:** Remove the previously created entries `"<key>" "<value>"`

```json
{
  "globalSettings": {
    "selfHosted": true,
    "identityServer": {
      "certificateThumbprint": "<your Identity certificate thumbprint with no spaces>"
    },
    "dataProtection": {
      "certificateThumbprint": "<your Data Protection certificate thumbprint with no spaces>"
    },
    "installation": {
      "id": "<your Installation Id>",
      "key": "<your Installation Key>"
    },
    "licenseDirectory": "<full path to licence directory>",
    "sqlServer": {
      "connectionString": "Server=localhost;Database=vault_dev;User Id=sa;Password=<your SA_PASSWORD>"
    }
  }
}
```

# After the first installation

**Note:** Make sure you followed [this guide](https://github.com/passcert-project/browser/blob/master/README.md) to start the bitwarden's browser extension. Only then will the below command work.

Open 4 terminals (I suggest [tmux](https://github.com/tmux/tmux/wiki)):

- navigate to `./browser` and run the command `npm run build:watch`
- navigate to `./server`
  - run the command `cd src/Identity` and then `dotnet restore; dotnet build; dotnet run`
  - run the command `cd src/Api` and then `dotnet restore; dotnet build; dotnet run`
  - run the command: `docker start <name of the DB container`

## Test the setup

Check the addresses:

- `http://localhost:33656/.well-known/openid-configuration`

- `http://localhost:4000/alive`

They should both be visible and respond. This means that the server is ready.

- If your test was successful, you can connect a GUI client to the dev environment by following the instructions here: [Change your client application's environment](https://bitwarden.com/help/article/change-client-environment/). If you are following this guide, you should only set the API Server URL and Identity Server URL to localhost:port and leave all other fields blank.
