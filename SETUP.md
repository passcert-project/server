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

There are 2 options for deploying your own SQL server.

### Without Docker

1. Install your own SQL server on localhost (e.g. SQL Express)
2. Right-click the SQL project in Visual Studio and click **Snapshot Project**. This will produce a .dacpac file containing the database schema
3. Use your preferred database management software (such as SQL Server Management Studio) to deploy a new database from the .dacpac file

### With Docker

1. Install [Docker and Docker Compose](https://bitwarden.com/help/article/install-on-premise/#install-docker-and-docker-compose) if you haven't already.

2. Install [Bitwarden's platform on docker](https://bitwarden.com/help/article/install-on-premise/#install-bitwarden)

- For the domain name, insert `localhost`
- Select `no` for the Let's Encrypt Certificate
- Copy the instalation id from the link specified in the above link
- Select `no` when prompted to use your own certificate
- Select `yes` for using and generating a self-signed certificate

3. Now run the command `./bitwarden.sh start`. This will give you the entire Bitwarden Server (not just the SQL server), but it is the quickest and easiest method to get what you need.

- Maybe run the command `docker ps` to see the status of each container

4. Now stop the execution with `./bitwarden.sh stop`

5. Open a terminal with elevated privileges and navigate to your `bwdata` install folder

- Might work without elevated privileges

6. Run the SQL Docker container with these arguments:

   ```bash
   docker run -d -e "ACCEPT_EULA=Y" -e "SA_PASSWORD=<set an SQL password here>" -p 1433:1433 --name mssql-dev \
   --mount type=bind,source="$(pwd)"/mssql/data,target=/var/opt/mssql/data \
   --mount type=bind,source="$(pwd)"/logs/mssql,target=/var/opt/mssql/log \
   --mount type=bind,source="$(pwd)"/mssql/backups,target=/etc/bitwarden/mssql/backups bitwarden/mssql
   ```

   **Note 1:** Beware the rules of the password. It has to be at least 8 characters and contain 3 of the 4 classes: _lower-case, upper-case, numbers, symbols_.

**Note 2:** you will need the `SA_PASSWORD` you set here for the connection string in your user secrets (see below).

The SQL server should be running smoothly. :tada:

# Server

## User Secrets

User secrets are a method for managing application settings on a per-developer basis. They are stored outside of the local git repository so that they are not pushed to remote.

User secrets override the settings in `appsettings.json` of each project. Your user secrets file should match the structure of the `appsettings.json` file for the settings you intend to override.

For more information, see: [Safe storage of app secrets in development in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets?view=aspnetcore-3.1).

### Editing user secrets - Visual Studio on Windows

Right-click on the project in the Solution Explorer and click **Manage User Secrets**.

### Editing user secrets - Visual Studio on macOS

Open a terminal and navigate to the project directory. Once there, initiate and create the blank user secrets file by running:

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

- I'm using macOs and my folder path is: `c:/bwtest/licenses`

**installation\_\_key** and **installation\_\_id**: request your own private Installation Id and Installation Key for self-hosting: https://bitwarden.com/host/.

## Example User Secrets file

This is an example user secrets file for both the Api and Identity projects.

**Note:** Possibly, it will be found in the folder `~/.microsoft/usersecrets/`

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
      "connectionString": "Data Source=localhost,1433;Initial Catalog=vault;Persist Security Info=False;User ID=sa;Password=<your SQL password>;MultipleActiveResultSets=False;Connect Timeout=30;Encrypt=True;TrustServerCertificate=True"
    }
  }
}
```

### Possible setup error in `src/Identity`

You may encounter an `Invalid licensing certificate` when running the command `dotnet run` in the project `src/Identity`, like this one:

```console
info: Microsoft.AspNetCore.DataProtection.KeyManagement.XmlKeyManager[0]
      User profile is available. Using '/Users/<username>/.aspnet/DataProtection-Keys' as key repository; keys will not be encrypted at rest.
info: IdentityServer4.Startup[0]
      Starting IdentityServer4 version 4.0.4+1b36d1b414f4e0f965af97ab2a7e9dd1b5167bca
crit: Microsoft.AspNetCore.Hosting.Diagnostics[6]
      Application startup exception
System.Exception: Invalid licensing certificate.
```

As a quick fix, navigate to [`src/Core/Services/Implementations/LicensingService.cs`](https://github.com/bitwarden/server/blob/df7a035d9bdbee40e019a596a1a7a66826db02f4/src/Core/Services/Implementations/LicensingService.cs#L49), line 49, and change that line to

```cs
var certThumbprint = !environment.IsDevelopment() ? "207E64A231E8AA32AAF68A61037C075EBEBD553F" :
                "‎B34876439FCDA2846505B2EFBBA6C4A951313EBE";
```

⚠️ Do not commit this change, as it might ruin the experience for others.

# After the first installation

**Note:** Make sure you followed [this guide](https://github.com/passcert-project/browser/blob/master/README.md) to start the bitwarden's browser extension. Only then will the below command work.

Open 4 terminals (I suggest [tmux](https://github.com/tmux/tmux/wiki)):

- navigate to `./browser` and run the command `npm run build:watch`
- navigate to `./server`
  - run the command `cd src/Identity` and then `dotnet restore; dotnet build; dotnet run`
  - run the command `cd src/Api` and then `dotnet restore; dotnet build; dotnet run`
  - run the command: `docker start mssql-dev`

## Test the setup

Check the addresses:

- `http://localhost:33656/.well-known/openid-configuration`

- `http://localhost:4000/alive`

They should both be visible and respond. This means that the server is ready.
