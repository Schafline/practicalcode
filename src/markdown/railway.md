# Chapter 15: Deploying Your Storefront{#deploying-your-storefront}

## 15.1 Introduction

Deploying a web app can get complicated, depending on the platform you
choose but for this tutorial we're aiming for a deployment process
that's as straightforward as possible while still choosing a solution
that fits our app's needs in terms of technology, scalability, security,
ease of setup and cost.

That's why, in this chapter, we'll use **Railway**, a modern hosting
platform designed to make deployment simple, fast and beginner‑friendly.

Railway gives you everything you need to get your app online without the
complexity of traditional cloud platforms. There are no pricing tiers to
decode, no resource groups to configure and no surprise charges waiting
in the shadows. You can create an account in minutes, deploy directly
from Visual Studio Code and your app will be live on a secure URL almost
immediately.

Most importantly, Railway works beautifully with the technologies we're
using in this book. Your Razor Pages app runs exactly as it does on your
machine and Railway provides a fully managed PostgreSQL database that
integrates seamlessly with Entity Framework Core.

During development we used SQLite because it's lightweight and requires
no setup but for production Railway gives us a reliable, cloud‑ready
PostgreSQL instance with zero configuration. You don't need to provision
servers, manage backups, or learn a new cloud database platform since
Railway handles everything automatically for us.

::: generalheading
★ Starting This Chapter
:::

Before beginning the steps in this chapter, follow the first two steps
of the chapter workflow described in [[Appendix
1](#how-to-use-the-chapter-snapshots)]{.hyperlink}:

1\. **Ensure your main branch is checked out**.\
**main** always points to the correct snapshot for the chapter you've
just completed.

2\. **Create a fresh working branch for this chapter**.\
This gives you a clean, isolated place to work through the chapter's
changes.

::: generalheading
Chapter Goals and What You'll Learn
:::

By the end of this chapter, you'll be able to:

☑  Deploy your Storefront application using a simple, beginner‑friendly
hosting platform

☑  Set up a Railway project and connect it to your local code

☑  Publish your Razor Pages app directly from Visual Studio Code using
the Railway CLI

☑  Configure production‑only settings such as environment variables and
your admin password hash

☑  Verify your deployment and access your live Storefront site

☑  Use Railway's built‑in logs and diagnostics to troubleshoot issues if
something goes wrong in production
<div class="figure">
::: callouttitle
ℹ Why This Book Uses Railway
:::

You might be wondering why we're not deploying to a traditional web host
like Hostinger or GoDaddy, or to a large cloud provider like Azure or
AWS. The reason is simple: this book is about learning Razor Pages and
building a real, working storefront, not about navigating
enterprise‑grade cloud services.
</div>
Most shared hosts don't support modern ASP.NET Core applications in a
clean, reliable way. And while platforms like Azure App Service are
powerful, they introduce a lot of concepts that aren't relevant to what
we're trying to achieve here: pricing tiers, resource groups, deployment
slots and a whole vocabulary of cloud‑specific features that can
distract from the core learning experience.

Railway, on the other hand, keeps things focused. It handles the
runtime, HTTPS (HTTPS ensures encrypted, secure communication between
browser and server), environment variables and deployment pipeline
automatically, without asking you to become a cloud architect. You get a
smooth, predictable deployment experience that lets us stay focused on
the application itself.

## 15.2 Install the Railway CLI

To deploy your Storefront directly from Visual Studio Code, we'll use
the Railway CLI. This is a small command‑line tool that handles
everything for you. Building your project, uploading it to Railway and
spinning up your live site. It's lightweight, easy to install and works
on any platform.

### 15.2.1 Install Node.js

The CLI is installed using **npm**, which comes bundled with Node.js.
Let's do a quick check to see if you have Node.js installed.

::: instructionsubtitle
Run the following command:
:::

::: sourcecode
    node -v
:::

If you see a version number after issuing the [node
-v]{.inlinecodefixed} command (for example,
[v16.17.0]{.inlinecodefixed}), you're good to go.

::: introsubsectionheading
If Node.js is not installed
:::

::: instructiontitle
Windows
:::

::: instructionsubtitle
Download the LTS version from:
:::

[[https://nodejs.org]{.hyperlink}](https://nodejs.org/)

If [node -v]{.inlinecodefixed} isn't recognised after installation,
restart your terminal or reinstall Node.js as some Windows setups don't
update the PATH correctly.

::: instructiontitle
Linux / macOS
:::

The recommended way to install Node.js on Linux and macOS is **nvm**
(Node Version Manager). It avoids permission issues and ensures npm
works correctly.

[Install nvm]{.instructionsubtitlechar}

::: sourcecode
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
:::

::: instructionsubtitle
Reload your shell
:::

::: sourcecode
    source ~/.bashrc   # or ~/.zshrc on macOS
:::

::: instructionsubtitle
Install the latest LTS version of Node
:::

::: sourcecode
    nvm install --lts
:::

::: instructionsubtitle
Verify
:::

::: sourcecode
    node -v
    npm -v
:::

Once Node.js is installed, the [npm]{.inlinecodefixed} command will be
available.

::: instructionsubtitle
Install the Railway CLI:
:::

::: sourcecode
    npm install -g @railway/cli
:::

This installs the CLI globally so you can use the
[railway]{.inlinecodefixed} command from any project.

## 15.3 Login to Railway

Now that the Railway CLI is installed, the next step is to sign in so
the CLI can connect to your Railway account. This only takes a moment.

::: instructionsubtitle
Open the terminal in Visual Studio Code and run:
:::

::: sourcecode
    railway login
:::

The CLI may display a short message about optional telemetry collection.
After that, it may ask whether it should open your browser to complete
the login:\
![](media/image67.png)

Press **Y** and your browser will open to Railway's login page as shown in Figure 15-1.
<div class="figure">
::: label
Figure 15-1: Railway Login Page
:::
![](media/image68.png)
</div>
[Click]{.instructionsubtitlechar} [Continue with
GitHub]{.heading12pttarget} [to sign in.]{.instructionsubtitlechar}

This is the simplest and most reliable option and because you already
created a GitHub account earlier in the book, you won't need to enter
verification codes or deal with session timeouts.

The first time you sign in with GitHub, you'll be taken to a GitHub
permissions screen asking whether Railway is allowed to access your
GitHub account. This is normal. Review the permissions and click to
continue.

If you sign in using **email**, Railway will first ask you to enter the
verification code sent to your inbox.

After you've successfully logged in, Railway will show a confirmation
screen asking which workspaces the CLI should be allowed to access as
shown in Figure 15-2.
<div class="figure">
::: label
Figure 15-2: Railway CLI Confirmation Screen
:::
![](media/image69.png)
</div>

[Choose All Workspaces and click Authorize]{.instructionsubtitlechar}.

At this point, Railway normally completes the login and shows an
**Authentication Successful** message. You can close the browser tab and
back in VS Code the terminal will confirm that you're now logged in and
display your Railway identity.

Occasionally, especially when signing in with **email**, Railway may
display a "session expired" message instead of completing the login.

If this happens:

- ::: listparagraph
  Close the browser tab
  :::

- ::: listparagraph
  Return to VS Code and the terminal will still show a "waiting for
  authentication" message
  :::

- ::: listparagraph
  Cancel it by pressing [Ctrl+C]{.inlinecodefixed} (or the macOS
  equivalent)
  :::

- ::: listparagraph
  Run [railway login]{.inlinecodefixed} again
  :::

You'll be redirected to the browser once more. This time, Railway
usually detects the previous attempt and immediately shows an
**Authentication Successful** message. Back in VS Code, the terminal
will confirm that you're now logged in and display your Railway
identity.

The CLI remembers your session, so you won't need to repeat this step
unless you sign out or switch machines.

## 15.4 Initialize Your Project

With the CLI installed and authenticated, the next step is to connect
your local Storefront project to a Railway project. This creates the
link that allows Railway to build and deploy your application.

**Run:**

::: sourcecode
    railway init
:::

Making sure you're inside the root folder of your Storefront application
in VS Code.

Railway will guide you through two quick prompts:

- ::: listparagraph
  **Select a workspace** Railway automatically selects your default
  workspace (usually called *My Projects*). There's nothing to change
  here, just press **Enter**
  :::

- ::: listparagraph
  **Project Name** Enter a name for your project, such as
  **Storefront**, or press **Enter** to let Railway generate a name
  automatically
  :::

Your terminal will look something like this:

![](media/image70.png)

Once you hit return, Railway creates a new project in your account and
links your local folder to it.

**To verify the link run:**

::: sourcecode
    railway status
:::

Before your first deployment, this will show something like:

![](media/image71.png)

The [Service:None]{.inlinecodefixed} line is expected; Railway only
creates a service after your first deployment.

At this point, your local Storefront app is officially connected to
Railway and ready to deploy.

## 15.5 Deploy Your Storefront

With your project linked to Railway, you're ready to deploy your
Storefront application. Railway makes this step refreshingly simple;
everything happens through a single command.

### 15.5.1 Create a .railwayignore file to prevent deployment timeouts

Before deploying your application, it's a good idea to create a
**.railwayignore** file in the root of your project. Railway uploads
everything in your project directory unless told otherwise and large
folders such as *bin/* and *obj/* can slow deployments down or even
cause them to time out. A **.railwayignore** file works just like a
**.gitignore** file: it tells Railway which files and folders it can
safely skip.

::: instructionsubtitle
Create a new file named .railwayignore and add the following entries:
:::

::: label
.railwayignore
:::

::: sourcecode
    bin/
    obj/
    *.user
    *.db
    *.sqlite
    *.mdf
    *.ldf
    .vscode/
    .idea/
:::

These folders contain build artefacts, IDE settings and local database
files that don't need to be uploaded to Railway. Excluding them keeps
your deployments fast and predictable.

[Important]{.instructionsubtitlechar}:

Do **not** add the *wwwroot/lib* folder to the ignore list. This folder
contains Bootstrap, jQuery and other client‑side libraries your
storefront needs in production. If it's excluded, your site will deploy
without these assets, leading to missing styling and JavaScript errors.

With this **.railwayignore** file in place, your deployments will be
smaller, faster and far less likely to hit timeouts.

### 15.5.2 Deploy your storefront with the Railway CLI

With your production settings in place and your **.railwayignore** file
keeping the deployment lean, you're ready to deploy your storefront to
Railway. The simplest way to do this is with the Railway CLI. It
packages your application, uploads it and triggers a new deployment in
one step.

::: instructionsubtitle
From the root of your project, run:
:::

::: sourcecode
    railway up
:::

On the **first** run of this command in a brand‑new project folder,
Railway does not prompt you for anything. It assumes this is a new
deployment and automatically:

- ::: listparagraph
  Builds your application in Release mode
  :::

- ::: listparagraph
  Compresses the published output
  :::

- ::: listparagraph
  Creates a new Railway project and service for you
  :::

- ::: listparagraph
  Uploads your files
  :::

- ::: listparagraph
  Triggers a deployment
  :::

- ::: listparagraph
  Streams the build logs in real time
  :::

This first deployment is intentionally friction‑free. You simply press
**Enter** and watch your storefront come to life on Railway.

On subsequent deployments, the CLI remembers which service this folder
is linked to and will ask you to confirm it before continuing. Just
press **Enter** to accept the default service name and Railway will
deploy your updated storefront:\
![](media/image72.png)

After you confirm, the CLI provides clear feedback:

- ::: listparagraph
  It tells you that your files have been compressed
  :::

- ::: listparagraph
  It shows that the upload has completed
  :::

- ::: listparagraph
  And when the build starts, it prints a **build log URL** you can open
  in your browser
  :::

Once those confirmations appear, Railway streams the full build output
directly into your terminal. This includes all compiler messages,
warnings, dependency restores and any other information produced during
the build. There can be quite a lot of output here and that's normal;
it's the same information you would see if you were building the project
locally.
<div class="figure">
::: callouttitle
⚙ The Railway Project Dashboard
:::

When you deploy for the first time, Railway automatically creates a
project for you. You can view and manage this project in the Railway
project dashboard:
[[https://railway.com/dashboard]{.hyperlink}](https://railway.com/dashboard)
</div>
The dashboard shows:

- ::: listparagraph
  Your project
  :::

- ::: listparagraph
  Your services
  :::

- ::: listparagraph
  Your deployments
  :::

- ::: listparagraph
  Your logs
  :::

- ::: listparagraph
  And any public domains you assign later
  :::

Whenever the CLI prints a link to a build log, it takes you directly to
a corresponding log view in this Railway dashboard, which makes it much
easier to examine errors or warnings in a clean, scrollable interface.

If anything goes wrong during the build or deploy steps, you can open
the build log URL printed by the CLI. This link takes you straight to
the corresponding log view in the Railway dashboard, which makes it much
easier to examine errors or warnings in a clean, scrollable interface.

Railway also sends a deployment status email to the address associated
with your account. This happens automatically for every deployment, so
even if you close your terminal or step away from your machine, you'll
still receive a confirmation that the deployment succeeded (or a
notification if it failed)

When the build and deploy steps complete successfully, the CLI prints a
clear **Deploy complete** message.
<div class="figure">
::: callouttitle
⚠️ After the deployment finishes
:::

You may notice something unexpected in the Railway dashboard: the
service will briefly show as **Running** and then switch to **Crashed**.
This is normal at this stage of the chapter.
</div>
Your application is still using [SQLite]{.strong}, which works perfectly
in development but cannot run inside Railway's production environment.
Railway containers use an ephemeral, read‑only filesystem, so SQLite
cannot create its database file. When the app starts and tries to query
the **Products** table, Entity Framework throws an exception and the
service exits.

This temporary crash is part of the normal deployment flow. In the next
section, we'll connect your storefront to a real PostgreSQL database and
the application will start successfully.

Any time you make changes to your project (whether it's code,
configuration, content, or any other project resource) you'll need to
run [railway up]{.inlinecodefixed} again to redeploy those changes.
Railway doesn't automatically rebuild your service when files change
locally; a new deployment is only triggered when you explicitly run the
command.
<div class="figure">
::: callouttitle
⚠️ When "Deploy complete" appears locally but no new deploy shows in the
Railway dashboard
:::

If the CLI reports **Deploy complete** but the Railway dashboard shows
no new deploy, your local folder is no longer linked to the correct
service.
</div>
The Railway CLI keeps a record of project links in a hidden
configuration file under your user profile (for example,
*C:\\Users\\\<username\>\\.railway\\config.json* ).

If the service ID stored there becomes stale (such as after deleting or
recreating a service in the dashboard) the CLI still knows your project
and environment but it no longer knows which service to update.

In this state:

- ::: listparagraph
  Running [railway status]{.inlinecodefixed} will show your project and
  environment
  :::

- ::: listparagraph
  But will list **Service: None**
  :::

- ::: listparagraph
  And [railway up]{.inlinecodefixed} will silently create a new service
  and deploy there instead
  :::

This is why the dashboard appears unchanged even though the CLI reports
success.

**To fix this**:

- ::: listparagraph
  Run [railway init]{.inlinecodefixed} inside your project folder
  :::

- ::: listparagraph
  Reselect your application service
  :::

  - ::: listparagraph
    Use the arrow keys to highlight the service that represents your app
    (e.g., storefront)
    :::

  - ::: listparagraph
    **Do not select** the PostgreSQL service, if listed.
    :::

- ::: listparagraph
  Confirm your choice.\
  The CLI updates its stored service ID. After relinking:
  :::

  - ::: listparagraph
    [railway status]{.inlinecodefixed} will show the correct service
    :::

  - ::: listparagraph
    Subsequent [railway up]{.inlinecodefixed} commands will deploy to
    the intended service
    :::

  - ::: listparagraph
    New deployments will appear in the dashboard as expected.
    :::

### 15.5.3 Make your storefront publicly accessible

At this stage, your deployment has completed successfully but the
application itself is not running yet. Because the service crashes
during startup (due to SQLite not being supported in Railway's
production environment), Railway cannot start the container or bind it
to a port.

You can still continue with the next step and generate a public domain
for your storefront but you may see a **502 Bad Gateway** when visiting
the URL. This is expected. In the next section, you'll connect your
application to PostgreSQL and once the service can start normally, the
domain will load your storefront as intended..

To make your application accessible on the internet, open your Railway
project dashboard and you should see your project represented as a tile
on a designer-like canvas, as shown in Figure 15-3.

Click on the small **\>\_** icon inside
the project tile. This icon represents the Storefront service and clicking it takes
you directly to the service configuration as shown in Figure 15-4. If you click the main body of the project tile instead, you’ll land on another canvas showing the services as individual tiles (although you will currently just have one). From there, click the Storefront service tile to reach the same place.
<div class="figure">
::: callouttitle
ℹ Railway UI Updates
:::

Railway's UI evolves regularly. Updated deployment and configuration
screenshots can be found at:
</div>
::: hyperlink
[[https://github.com/Schafline/Storefront/tree/main/external-ui-updates/railway]{.hyperlink}](https://github.com/Schafline/Storefront/tree/main/external-ui-updates/railway)
:::
<div class="figure">
::: label
Figure 15-3: Railway Dashboard
:::

![](media/image73.png)
</div>
By default, the **Deployments** tab is selected when you first open the service configuration and this displays the history of your deployments as shown in Figure 15-4.

At the top of this list, you'll see the crashed deployment you just
triggered with [railway up]{.inlinecodefixed}, along with its status,
timestamp and a link to the full build logs. This is the best place to
confirm what happened and review the logs if anything went wrong..

From the Service configuration, click the **Settings** tab. This is where Railway keeps all service level configuration
options.

Scroll down to the **Networking** section. This is where Railway keeps
the domain controls and is shown in Figure 15-5.

::: instructionsubtitle
Click Generate Domain
:::

To create a public URL for your storefront. Railway will prompt you for
a port number; enter **8080**, which is the port your ASP.NET Core
application listens on inside the container.
<div class="figure">
::: label
Figure 15-4: Railway Service Configuration - Deployments View
:::

![](media/image74.png)
</div>
After you confirm the port, Railway assigns a unique domain
automatically and it becomes active immediately.
<div class="figure">
::: label
Figure 15-5: Railway Service Configuration - Settings View
:::

![](media/image75.png)
</div>
Once the domain is generated, it appears in the list below the button,
similar to the screenshot below:
<div class="figure">
::: label
Domain Generation Success
:::

![](media/image76.png)
</div>
You can click on the generated domain to open your storefront in the
browser. This is the same URL you'll share with customers, testers, or
anyone who needs access to your live site.

If you ever redeploy or update your application, the domain remains the
same, only the underlying deployment changes. This makes it easy to
iterate on your storefront without breaking links or updating bookmarks.

At this point, your storefront has a public URL. It's normal to see a
404 right now and the site will come online as soon as your first
successful deployment completes.

The final part of the deployment config is giving it a production‑grade
database, so next we'll connect your application to Railway's managed
Postgres service.

## 15.6 Add a Production‑Ready PostgreSQL Database

Your application is deployed but it can't start yet because SQLite isn't
supported in Railway's production environment. To make the storefront
run reliably in the cloud, we'll switch to **PostgreSQL** which is a
fully managed, production‑ready database that works seamlessly with
Entity Framework Core.

In this section, we'll:

- ::: listparagraph
  Create a PostgreSQL database in Railway
  :::

- ::: listparagraph
  Add the connection string to the service configuration
  :::

- ::: listparagraph
  Update the application to use PostgreSQL in Production
  :::

- ::: listparagraph
  Apply the EF Core migrations
  :::

Once PostgreSQL is in place, your storefront will have a durable,
scalable production database that survives restarts, redeployments and
traffic spikes.

### 15.6.1 Creating a PostgreSQL database in Railway

Railway makes it easy to provision a managed PostgreSQL database but the
UI can be a little confusing the first time you see it. Databases are
created inside a service, not at the project level, so you must make
sure you're looking at the correct view before adding one.

To create your database:

- ::: listparagraph
  **Open the Railway project dashboard**.\
  You should see your Storefront project tile on the canvas (Figure 15-4).
  :::

- ::: listparagraph
  **Click the \>\_ icon on the Storefront project tile.**\
  This opens the service configuration (Figure 15-5).
  :::

- ::: listparagraph
  **Close the service configuration (the Deployments, Metrics, Variables tabs etc.).**\
  This hides the controls you need. Click the **X** in the top right corner to close it.\

  Now you should see your Storefront service tile and an Add button. The button appears at the top right of the canvas, as shown in Figure 15-6. This is the correct place to add resources to your project.
  :::

<div class="figure">
::: label
Figure 15-6: Storefront Service Tile
:::
![](media/image77.png)
</div>

- ::: listparagraph
  **Click the Add button**.
  :::

- ::: listparagraph
  **Choose Database from the list of options**.
  :::

- ::: listparagraph
  **Select PostgreSQL as the database type.**\
  Railway will immediately provision a new Postgres service with sensible defaults and a new Postgres service tile should be added to the as shown in Figure 15-7.
  :::
<div class="figure">
::: label
Figure 15-7: Postgres Service Tile
:::
  ![](media/image78.png)
</div>

### 15.6.2 Configure the Postgres connection string

- ::: listparagraph
  **Click on the Postgres service tile to open the Postgres service configuration.**
  To connect the storefront to Postgres, the application needs access to the database connection string. Railway exposes this as an environment variable on the Postgres service and we can reference it directly from the storefront service without copying or pasting anything.
  :::

- ::: listparagraph
  **Open the Variables tab for the Postgres service**
  You'll see two relevant entries:
  :::

  - ::: listparagraph
    [DATABASE_URL]{.inlinecodefixed}: The internal connection string used
    by other services in the same project
    :::

  - ::: listparagraph
    [DATABASE_PUBLIC_URL]{.inlinecodefixed}: A public proxy connection
    string for external tools
    :::

For the storefront, we'll use [DATABASE_URL]{.inlinecodefixed}.

The next step is to make this value available to your storefront service
so the deployed application can read it at runtime.

The storefront service now needs access to the Postgres connection
string so it can connect to the database in production. Railway makes
this easy by allowing services inside the same project to reference each
other's environment variables directly.

To configure the storefront service:

- ::: listparagraph
  **Close the Postgres service configuration and click the Storefront tile **\
  You should see the familiar tabs: *Deployments*, *Variables*,
  *Settings* and so on.
  :::

- ::: listparagraph
  **Select the Variables tab**\
  This is where your application's environment variables live.
  :::

- ::: listparagraph
  **Add a new variable named** [DATABASE_URL]{.inlinecodefixed}\
  Click the New Variable button to open the variable editor. In the Name
  field, start typing [DATABASE_URL]{.inlinecodefixed}. Railway detects
  that the Postgres service already exposes a variable with this name
  and it appears in a dropdown list.\
  Select it and Railway automatically fills the value field, as shown in Figure 15-8.
  :::
  <div class="figure">
  ::: label
  Figure 15-8: Storefront Service Variables
  :::

  ![](media/image79.png)
  </div>
- ::: listparagraph
  **Click Add to save the new variable**\
  Railway stores the change immediately but it does not redeploy the
  service yet. A **Deploy** button will appear on the canvas to indicate 
  that the configuration has changed and a new deployment is required, see Figure 15-9.
  :::

<div class="figure">
::: label
Figure 15-9: Service Link and Pending Deploy
:::

![](media/image80.png)
</div>

- ::: listparagraph
  **Click Deploy to apply the updated environment to the running
  application**
  :::

When you link the [DATABASE_URL]{.inlinecodefixed} variable to the 
Postgres service, Railway shows this relationship visually on the canvas. 
The storefront service tile now displays an arrow pointing to the 
Postgres service tile, indicating that the application depends on the 
database and receives its connection string from it.

If you ever see an **edited** label on the storefront tile, it simply
means the service configuration has changed since the last deployment;
for example, after adding or updating an environment variable. It isn't
an error; it just indicates that a redeploy is needed for the change to
take effect.

At this point, your storefront service has access to the PostgreSQL
connection string at runtime. The application won't use it yet because
we still need to update the ASP.NET Core configuration so it switches
from SQLite to PostgreSQL when running in production.

## 15.7 Update the Application to Use PostgreSQL in Production

The storefront service now receives a PostgreSQL connection string
through the [DATABASE_URL]{.inlinecodefixed} environment variable but
the application itself still needs to be updated so it can run correctly
against PostgreSQL in production.

At the moment, the app is configured to use SQLite by default, which is
perfect for local development but not suitable for a cloud‑hosted
production environment.

To complete the transition, we need to:

- ::: listparagraph
  Regenerate the EF Core migrations for PostgreSQL
  :::

- ::: listparagraph
  Update the design‑time factory so EF Core can create a
  PostgreSQL‑backed context
  :::

- ::: listparagraph
  Modify **Program.cs** so the deployed application selects the correct
  provider and applies migrations automatically on startup
  :::

Before updating the application code, the project needs the EF Core
provider for PostgreSQL. This package contains the Npgsql database
provider and the tools EF Core uses to generate PostgreSQL‑specific
migrations.

::: instructionsubtitle
Install it by running the following command in the *storefront* project
folder:
:::

::: sourcecode
    dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL -v 9.0.0
:::

With the provider installed, EF Core can create migrations that target
PostgreSQL and the application can use the [UseNpgsql]{.inlinecodefixed}
method when running in production.

### 15.7.1 Regenerate EF Core migrations for PostgreSQL

Before the application can run against PostgreSQL in production, the EF
Core migrations need to be regenerated. The existing migrations were
created using SQLite and each database provider generates slightly
different SQL.

To ensure the schema matches what PostgreSQL expects, we'll create a
fresh initial migration using the PostgreSQL provider.

This requires a small update to the design‑time
[ShopContextFactory]{.inlinecodefixed} and a temporary switch to
Production mode while generating the migration locally.

### 15.7.2 Update ShopContextFactory

EF Core uses the [IDesignTimeDbContextFactory]{.inlinecodefixed} when
running CLI commands such as [dotnet ef migrations
add]{.inlinecodefixed}. Because migrations must match the database
provider they target, the factory needs to select SQLite when you're
working locally and PostgreSQL when you temporarily switch your
environment to Production to generate PostgreSQL‑compatible migrations.

To support this, update the [CreateDbContext]{.inlinecodefixed} method
in **ShopContextFactory.cs** so it chooses the correct provider based on
the current environment and reads the [DATABASE_URL]{.inlinecodefixed}
connection string when running in Production.

::: introsubsectionheading
Inside CreateDbContext delete this existing code block
:::

::: sourcecode
        var optionsBuilder =
        new DbContextOptionsBuilder
        <ShopContext>();

        optionsBuilder.UseSqlite
        ("Data Source=shop.db");
:::

This existing block creates a
[DbContextOptionsBuilder]{.inlinecodefixed} and hard‑codes SQLite and
the logic is no longer needed.

You're about to replace it with an environment‑aware version that uses
SQLite in development and PostgreSQL in production.

::: introsubsectionheading
Update CreateDbContext with the following added code
:::

::: sourcecode
      public ShopContext CreateDbContext(
        string[] args)
      {
    //---ADDED CODE---
        var config = new ConfigurationBuilder()
          .AddJsonFile("appsettings.json")
          .AddJsonFile("appsettings.Development.json",
           optional: true)
          .AddEnvironmentVariables()
          .Build();

        var env =
          Environment.GetEnvironmentVariable(
          "ASPNETCORE_ENVIRONMENT");

        var optionsBuilder =
          new DbContextOptionsBuilder<ShopContext>();

        if (env == "Production")
        {
          var connectionString =
            Environment.GetEnvironmentVariable(
            "DATABASE_URL");

          optionsBuilder.UseNpgsql(connectionString);
        }
        else
        {
          // Use SQLite in development
          var sqlite =
            config.GetConnectionString(
           "ShopDbConnection");

          optionsBuilder.UseSqlite(sqlite);
        }
    //---END ADDED CODE---
        return new ShopContext(
          optionsBuilder.Options);
      }
:::

This version loads configuration, checks the current environment and
selects the correct database provider.
<div class="figure">
::: callouttitle
ℹ What Changed?
:::

The [ShopContextFactory]{.inlinecodefixed} now:
</div>
- ::: listparagraph
  Selects SQLite when the environment is Development
  :::

- ::: listparagraph
  Selects PostgreSQL when the environment is Production
  :::

- ::: listparagraph
  Reads the [DATABASE_URL]{.inlinecodefixed} environment variable when
  generating migrations
  :::

- ::: listparagraph
  Uses [NpgSql()]{.inlinecodefixed} instead of
  [UseSqlite()]{.inlinecodefixed} when running EF Core commands in
  Production mode
  :::

This ensures that EF Core generates PostgreSQL‑compatible migrations
when you temporarily switch your local environment to Production.
Without this change, EF Core would continue producing SQLite‑specific
SQL, which cannot be used on Railway.

### 15.7.3 Regenerate the Migrations

Follow these steps to produce a clean PostgreSQL migration:

- ::: listparagraph
  **Delete the existing *Migrations* folder**\
  These files contain SQLite‑specific SQL and cannot be reused
  :::

::: instructiontitle
Windows(PowerShell)
:::

::: instructionsubtitle
Run the following command to temporarily set the environment to
Production:
:::

::: sourcecode
    $env:ASPNETCORE_ENVIRONMENT=Production
:::

This ensures that the factory selects the PostrgeSQL provider.

::: instructionsubtitle
Run the following command to provide a temporary DATABASE_URL:
:::

::: sourcecode
    $env:DATABASE_URL=postgres://fake:fake@localhost:5432/fake
:::

EF Core needs a connection string even though it won't connect to a real
database during migration generation**.**

::: instructiontitle
Linux/macOS (bash, zsh)
:::

::: instructionsubtitle
Run the following command to temporarily set the environment to
Production:
:::

::: sourcecode
    export ASPNETCORE_ENVIRONMENT=Production
:::

::: instructionsubtitle
Run the following command to provide a temporary DATABASE_URL:
:::

::: sourcecode
    export DATABASE_URL=postgres://fake:fake@localhost:5432/fake
:::
<div class="figure">
::: callouttitle
⚠️ Important: Keep your terminal open
:::

The environment variables you set earlier
([ASPNETCORE_ENVIRONMENT]{.inlinecodefixed} and
[DATABASE_URL]{.inlinecodefixed}) are **temporary**. They only exist in
the terminal session where you created them.
</div>
If you close VS Code or the terminal window, these variables disappear.

When that happens, EF Core silently switches back to **Development
mode**, uses **SQLite** and generates a **SQLite‑based migration**.

If you deploy that migration to Railway, the deployment will fail (even
though if PostgreSQL configuration is correct) because the migration
snapshot no longer matches the PostgreSQL model used in Production.

::: introsubsectionheading
Before generating the migration, run this quick sanity check:
:::

::: instructiontitle
Windows(PowerShell)
:::

::: instructionsubtitle
Check ASPNETCORE_ENVIRONMENT
:::

::: sourcecode
    $env:ASPNETCORE_ENVIRONMENT
:::

::: instructionsubtitle
Check DATABASE_URL
:::

::: sourcecode
    $env:DATABASE_URL
:::

You should see:

- ::: listparagraph
  Production
  :::

- ::: listparagraph
  The temporary PostgreSQL URL you set earlier
  :::

If either command prints nothing, the variables have been lost. Set them
again before continuing.

::: instructiontitle
Linux/macOS (bash, zsh)
:::

::: instructionsubtitle
Check ASPNETCORE_ENVIRONMENT
:::

::: sourcecode
    echo $ASPNETCORE_ENVIRONMENT
:::

::: instructionsubtitle
Check DATABASE_URL
:::

::: sourcecode
    echo $DATABASE_URL
:::

You should see:

- ::: listparagraph
  Production
  :::

- ::: listparagraph
  The temporary PostgreSQL URL you set earlier
  :::

If either command prints nothing, the variables have been lost. Set them
again before continuing.

::: introsubsectionheading
Generate a fresh initial migration
:::

::: instructionsubtitle
Run this command inside the storefront project folder:
:::

::: sourcecode
    dotnet ef migrations add InitialCreate
:::

**Important:** Keep these temporary environment variables in place for
now. Your application will continue running in "Production mode" locally
until the end of this chapter. This is intentional and allows you to
complete the deployment steps using the PostgreSQL migration you just
generated.

With this regenerated migration, EF Core now produces
PostgreSQL‑compatible SQL. When the application starts on Railway, it
will apply this migration automatically and create the correct schema in
your production database**.**
<div class="figure">
::: callouttitle
⚠️ Subsequent Deployments
:::

After this first deployment, you can safely redeploy the application
from your normal branch without recreating the PostgreSQL migration, 
even after the original SQLite migrations are restored. Railway keeps
your existing PostgreSQL database and does not need the migration to run
again.
</div>
If you later change your data model and need to update the Production
schema, create a new migration with a descriptive name (for example,
[AddOrderStatus]{.inlinecodefixed}) and then deploy the application
normally using [railway up]{.inlinecodefixed}. Railway will detect the
new migration and apply it to the existing PostgreSQL database.

Do not create another [InitialCreate]{.inlinecodefixed} migration since
EF Core would attempt to apply it on top of the existing tables, which
would cause the deployment to fail.

### 15.7.4 Update Program.cs to select the correct provider

ASP.NET Core reads configuration in layers. Environment variables
override **appsettings.json** and the hosting environment
(**Development**, **Production**, etc.) determines which code path runs.
Railway sets [ASPNETCORE_ENVIRONMENT=Production]{.inlinecodefixed} by
default, which allows the application to choose the appropriate database
provider at runtime, just as the design‑time factory selects the correct
provider when generating migrations.

To support that switch, the application also needs to understand the
connection string format that Railway provides. Railway supplies the
PostgreSQL connection as a single URL in the form:

*postgres://user:pass@host:port/dbname*.

EF Core's PostgreSQL provider ([Npgsql]{.inlinecodefixed}) doesn't
accept this URL format directly; it expects a keyword/value connection
string such as:

[Host=\...;Port=\...;Username=\.....]{.inlinecodefixed}

To bridge the two formats, we parse the URL and rebuild it into the
format Npgsql expects. This is a common pattern when deploying .NET apps
to cloud platforms that use Heroku‑style connection strings.

Before adding the new environment‑aware configuration, you will remove
the existing [AddDbContext]{.inlinecodefixed} registration that always
uses SQLite in **Program.cs**.

::: introsubsectionheading
Remove this block
:::

::: sourcecode
    builder.Services.AddDbContext<ShopContext>(
      options =>
      {
        options.UseSqlite(
          builder.Configuration
            .GetConnectionString(
              "ShopDbConnection"));
      });
:::

This block must be deleted, because the updated version replaces it
entirely.

::: introsubsectionheading
Add the following code enclosed by the ADDED CODE comments
:::

::: sourcecode
    var culture = new CultureInfo("en-US");
    CultureInfo.DefaultThreadCurrentCulture = culture;
    CultureInfo.DefaultThreadCurrentUICulture = culture;
    //---ADDED CODE---
    var environment = builder.Environment;
    if (environment.IsDevelopment())
    {
      builder.Services.AddDbContext<ShopContext>(
        options =>
        {
          options.UseSqlite(
            builder.Configuration
              .GetConnectionString(
                "ShopDbConnection"));
        });
    }
    else
    {
      var rawUrl =
        Environment.GetEnvironmentVariable(
          "DATABASE_URL");
      var uri = new Uri(rawUrl);
      var userInfo = uri.UserInfo.Split(':');
      var connectionString =
        $@"Host={uri.Host};Port={uri.Port};
        Username={userInfo[0]};Password={userInfo[1]};
        Database={uri.AbsolutePath.TrimStart('/')};
        SSL Mode=Disable;Trust Server Certificate=true";

      builder.Services.AddDbContext<ShopContext>(
        options =>
          options.UseNpgsql(
            connectionString,
            npgsqlOptions =>
            {
              npgsqlOptions
                .EnableRetryOnFailure();
            }));
    }
    //---END ADDED CODE---
    builder.Services.AddAuthentication(
        CookieAuthenticationDefaults
            .AuthenticationScheme)
:::

This version checks the current environment and configures the correct
database provider: SQLite in development, PostgreSQL in production.
<div class="figure">
::: callouttitle
ℹ What Changed?
:::

This update adds the runtime logic that mirrors the design‑time factory:
</div>
- ::: listparagraph
  **Development** → the app uses SQLite
  :::

- ::: listparagraph
  **Production** → the app uses PostgreSQL
  :::

The code now:

- ::: listparagraph
  Reads the [DATABASE_URL]{.inlinecodefixed} environment variable
  :::

- ::: listparagraph
  Parses the Heroku‑style URL into the keyword/value format Npgsql
  expects
  :::

- ::: listparagraph
  Configures EF Core with [UseNpgsql()]{.inlinecodefixed} when running
  on Railway
  :::

- ::: listparagraph
  Keeps your local development environment unchanged
  :::

When the application starts in production, ASP.NET Core builds the
service container and selects the PostgreSQL provider because Railway
sets [ASPNETCORE_ENVIRONMENT=Production]{.inlinecodefixed}. The
connection string comes from the [DATABASE_URL]{.inlinecodefixed} you
linked earlier, which the application now parses into the format Npgsql
expects.

Your local development environment continues using SQLite, so you can
work with a lightweight database while the deployed app uses PostgreSQL
automatically.

### 15.7.5 Redeploying and verifying the production database

To apply the updated configuration and automatic migration logic,
redeploy the application to Railway.

[From the root of your project, run]{.instructionsubtitlechar}:

::: sourcecode
    railway up
:::

Railway builds and publishes the updated storefront service, then starts
the application in the Production environment. Because
[ASPNETCORE_ENVIRONMENT]{.inlinecodefixed} is set to
[Production]{.inlinecodefixed}, the app uses the PostgreSQL provider and
connects using the [DATABASE_URL]{.inlinecodefixed} you linked earlier.

On startup, the [Database.Migrate()]{.inlinecodefixed} call checks for
any pending migrations and applies them automatically. If this is the
first deployment, EF Core creates all the tables in the Railway
PostgreSQL database.

You can confirm this in the Railway logs: the application will report
that it is using the Npgsql provider and applying migrations if needed.

Once the service is running, open the deployed URL and interact with the
storefront to verify that the database is working as expected.

## 15.8 Setting Production Configuration Values

The deployed application needs several configuration values that should
not be stored in source control. ASP.NET Core reads these from
environment variables in Production and Railway provides an easy way to
manage them. To keep things simple and avoid confusion between JSON
files and deployment environments, we'll store all of our Production
configuration in environment variables rather than using an
**appsettings.Production.json** file.

### 15.8.1 Required production environment variables

Before continuing, add the following environment variables to your
Railway service. Just as you did when adding the PostgreSQL connection
string in [15.6.2](#configure-the-postgresql-connection-string), you'll
add each of these keys in the **Variables** tab of your Railway service.
Railway won't populate these values automatically; you'll enter them
yourself based on the settings you created earlier in the book.

These keys must be entered exactly as shown (**no leading or trailing
spaces!**)or ASP.NET Core will not read them. For now, just add the keys
themselves. We'll go through the correct values for each one in the
following subsections.

- ::: listparagraph
  Admin\_\_PasswordHash
  :::

- ::: listparagraph
  PayPal\_\_SandboxClientId
  :::

- ::: listparagraph
  Email\_\_ApiKey
  :::

- ::: listparagraph
  Email\_\_From
  :::

- ::: listparagraph
  Email\_\_FromName
  :::

These variables replace the Production settings you used during
development. Railway stores them securely and makes them available to
your application when it runs in the Production environment.
<div class="figure">
::: callouttitle
⚠️ Warning
:::

Railway does not trim whitespace and even an invisible space will
prevent ASP.NET Core from reading the value.
</div>
### 15.8.2 Setting the values for each key

Now that the keys are in place, let's add the correct values for each
one. The next sections explain where each value comes from and how to
retrieve it.

Two of these values were already created during development and stored
securely using **user secrets**:

- ::: listparagraph
  PayPal\_\_SandboxClientId
  :::

- ::: listparagraph
  Email\_\_ApiKey
  :::

If you need to retrieve these values so you can also add them to Railway
for Production, you can list them at any time by running:

::: sourcecode
    dotnet user-secrets list
:::

Other configuration values were added to your
**appsettings.Development.json** file earlier in the book. If you want
to use those same values in Production, you can simply copy them from
that file into Railway's Variables tab.

::: introsubsectionheading
Admin login
:::

Earlier in the book you generated two password hashes: one for
development and a second one to use in Production.

[Add the Production hash under the following
key]{.instructionsubtitlechar}:

- ::: listparagraph
  [Admin\_\_PasswordHash]{.inlinecodefixed}
  :::

If you no longer have the Production hash, you can temporarily reuse the
development hash instead. You'll find it in your
**appsettings.Development.json** file. This will allow the application
to run in Production but it isn't recommended as using the same password
in both environments reduces security. If possible, generate a fresh
Production‑only hash and use that instead.

::: introsubsectionheading
Paypal
:::

The PayPal sandbox client ID must also be supplied via environment
variables.

::: instructionsubtitle
Enter the value you retrieved from user secrets under:
:::

- ::: listparagraph
  [PayPal\_\_SandboxClientId]{.inlinecodefixed}
  :::

This value is bound to your existing [PayPalOptions]{.inlinecodefixed}
class at startup.

::: introsubsectionheading
Email (Brevo)
:::

The values for [Email\_\_From]{.inlinecodefixed} and
[Email\_\_FromName]{.inlinecodefixed} come directly from the Email
section of your **appsettings.Development.json** file (unless you wish
to specify different values for production).

[Add values for these keys]{.instructionsubtitlechar}:

- ::: listparagraph
  [Email\_\_From]{.inlinecodefixed}
  :::

- ::: listparagraph
  [Email\_\_FromName]{.inlinecodefixed}
  :::

- ::: listparagraph
  E[mail\_\_ApiKey]{.inlinecodefixed}
  :::

The value for [Email\_\_ApiKey]{.inlinecodefixed} can be found in the
corresponding key in user secrets which you retrieved earlier in
[15.8.2](#setting-the-values-for-each-key). If you no longer have it,
simply go back and run the command detailed in that section again.

After adding all of the required keys, your **Variables** panel should 
appear as shown in Figure 15-10. Each key appears in the list with its
value stored securely (Railway masks the actual values with asterisks)
and they are now active configuration entries for the service. This
gives you a quick way to confirm that all production only settings are
in place before restarting the deployment.
<div class="figure">
::: label
Figure 15-10: Storefront Service Variables
:::

![](media/image81.png)
</div>

Once the keys have been added, Railway will display a banner indicating
that configuration has changed.

[Click the]{.instructionsubtitlechar} [Deploy]{.heading12pttarget}
[button in the dashboard]{.instructionsubtitlechar}

So the application restarts and picks up the new settings.

After the deployment restarts, your production environment is fully
configured to send order confirmation emails through Brevo.

With the email settings complete, the deployment process is finished and
the application is ready for a full end‑to‑end check in its live
environment.

## 15.9 ⚡Try it Out

With the application deployed and your production environment
configured, let's run through a few quick checks to confirm everything
is working as expected.

These steps ensure that:

- ::: listparagraph
  Your PostgreSQL database is connected
  :::

- ::: listparagraph
  Your environment variables are being read correctly
  :::

- ::: listparagraph
  The admin area is secured
  :::

- ::: listparagraph
  The public store behaves the same way in production as it did locally
  :::

- ::: listparagraph
  **Visit your live site**
  :::

  ::: numberedlistparagraphaleftaligned
  Open your Railway deployment URL in the browser. You should see the
  storefront homepage.
  :::

- ::: listparagraph
  **[Log in to the admin area]{.listparagraphchar}**\
  Navigate to */Admin* and sign in using the admin credentials you set
  in your production environment variables.
  :::

- ::: listparagraph
  **Verify and edit seeded products**\
  Because the PostgreSQL database was seeded on first run, you should
  already see the two sample products (tee shirt and sneakers) in the
  products page. Use the Edit link on one of these products to confirm
  that the admin UI is working and that changes are saved correctly.
  :::

- ::: listparagraph
  **Check the public store**\
  Return to the homepage and confirm that the sample products (tee shirt
  and sneakers) appear in the storefront. If you edited a product in the
  admin area, make sure those changes are reflected here. Click through
  to a product detail page to ensure everything loads correctly and the
  PostgreSQL‑backed catalogue is working as expected.
  :::

- ::: listparagraph
  **Test the shopping cart**\
  Add a product to the cart and confirm that the cart updates as
  expected.
  :::

  - ::: listparagraph
    The checkout flow is still using the same PayPal sandbox account you
    configured during development, so you can safely complete the entire
    process; including signing in with a sandbox buyer account,
    approving the payment and being redirected back to your site's order
    confirmation page. No real money is involved.
    :::

  - ::: listparagraph
    Switching to live payments requires a PayPal Business account and
    live API credentials, which is beyond the scope of this book. If you
    decide to go live in the future, PayPal's developer documentation
    explains how to generate live client IDs, configure return URLs and
    enable live webhooks.
    :::

- ::: listparagraph
  **Look up your order using the verification code.**\
  After completing the checkout flow, you'll receive a verification code
  in two places:
  :::

  - ::: listparagraph
    On the order confirmation page and
    :::

  - ::: listparagraph
    In the order confirmation email sent to the address you entered at
    checkout
    :::

    ::: listparagraph
    Use either version of the code. Return to the homepage, open the
    Order Lookup page and enter the verification code to retrieve your
    order.
    :::

    ::: listparagraph
    Confirm that the correct order details are displayed. This verifies
    that the application is successfully reading and writing order data
    to your PostgreSQL database in Production and that email delivery is
    functioning correctly.
    :::

- ::: listparagraph
  **Review the order in the admin area**\
  Open the admin orders page at *LOCALHOST_URL/Admin/Orders*.\
  You should see the order you just placed listed with a status of
  **Paid**. This confirms that the checkout flow successfully created
  the order in your PostgreSQL database and that the admin interface can
  read it correctly.
  :::

- ::: listparagraph
  **Click Edit on the order and change its status to Pending**\
  To verify that updates are saved as expected.
  :::

<div class="figure">
::: callouttitle
⚠️ If something goes wrong after returning from PayPal
:::

If the PayPal workflow doesn't return to your site, or if the
confirmation page appears but the order isn't saved or the email isn't
sent, the first place to look is the application logs in Railway.
</div>
Any exception thrown during the PayPal callback or the order‑processing
step will be written to the logs using
[Console.Error.WriteLine]{.inlinecodefixed} and this will tell you
exactly what failed.

You can view them in two places:

- ::: listparagraph
  **Observability → Error Logs**\
  The **Observability Error Logs** view shows a live or time filtered
  stream of recent log output. It's ideal when you want to watch the
  checkout flow while testing it.
  :::
  <div class="figure">
  ::: label
  Figure 15-11: Railway Project Dashboard: Observability Error Logs
  :::

  ![](media/image82.png)
  </div>
- ::: listparagraph
  The **Observability** menu item is accessible from the left-hand menu
  of your project dashboard and its icon contains three vertical lines.
  You can see this selected in Figure 15-11.\
  \
  If this is your first time opening the Logs Stream, the panel may
  appear empty. Railway will offer an option to **Start with a simple
  dashboard**. Select this option and the relevant log groups will
  appear; including **Disk Usage**, **Memory Usage** and **Error
  Logs**.\
  \
  Once the dashboard is created, you'll see live log entries as they
  occur. In Figure 15-11, you can see an error logged during the
  Brevo email operation.
  :::

- ::: listparagraph
  **Logs (with filters)**\
  Railway also provides a dedicated Logs page where you can filter
  messages using simple search syntax and it's menu item appears
  directly below the **Observability** icon selected in the screenshot,
  above. This is useful when you want to find a specific error without
  scrolling through the full log stream. For example:\
  [\@service:\"Storefront\" AND \@level:error]{.inlinecodefixed} --
  shows only errors in the Storefront service.
  :::

## 15.10 Reset Your Local Environment Back to Development Mode

During the migration‑generation steps earlier in this chapter, you
temporarily switched your machine into Production mode and created a
temporary branch so EF Core could generate PostgreSQL‑compatible
migrations without affecting your main development workflow.

Now that your application is deployed and verified on Railway, you can
safely return your local environment to its normal Development setup.

- ::: listparagraph
  **Switch back to your main development branch**\
  Follow the same workflow described in [[Appendix
  1](#how-to-use-the-chapter-snapshots)]{.hyperlink} (steps 4--6), not
  forgetting to delete the contents of your Migrations folder. After
  that, reset your main branch to the chapter‑15 snapshot, which
  contains the final version of the code used throughout this book.\
  \
  You should not need to delete **shop.db** as we haven't generated any
  new migrations in the development environment since chapter 13, so
  your database and model are still in sync.\
  \
  This ensures that:
  :::

  - ::: listparagraph
    Your local database returns to SQLite
    :::

  - ::: listparagraph
    Your original SQLite migrations are restored
    :::

  - ::: listparagraph
    All Chapter 15 code changes (including the
    [ShopDbContextFactory]{.inlinecodefixed} and environment switching
    logic) are preserved
    :::

  - ::: listparagraph
    Your main branch ends in the correct final state
    :::

- ::: listparagraph
  **Clear the temporary environment variables**\
  Because the environment variables were set only in the current
  terminal session, you can simply close the terminal window. When you
  open a new terminal, those temporary variables will be gone and your
  environment will automatically return to Development.
  :::

- ::: listparagraph
  **Run the application locally**\
  Start the application again and you should now see your familiar
  SQLite‑backed development store, exactly as it was before the
  deployment steps.
  :::

::: introsubsectionheading
Why this works
:::

::: instructionsubtitle
Your deployed application on Railway continues using PostgreSQL in
Production, while your local environment returns to SQLite. This gives
you the best of both worlds:
:::

- ::: listparagraph
  Fast, lightweight development locally
  :::

- ::: listparagraph
  Robust, cloud‑hosted PostgreSQL in Production
  :::

  ::: numberedlistparagraphaleftaligned
  With your environment reset, you can continue building features safely
  without touching your live data.
  :::

You now have a fully functioning, database‑backed online store running
in a real production environment. Your application can authenticate
administrators, process checkouts through PayPal's sandbox, send order
confirmation emails through Brevo and persist data in a managed
PostgreSQL database; all deployed automatically through Railway.

This is a major milestone. You've taken a local development project and
turned it into a live, cloud‑hosted application with a complete
end‑to‑end workflow.

In the next chapter, we'll step back and reflect on what you've built,
explore where you can take it next and look at the kinds of improvements
and extensions that real‑world storefronts often grow into.
