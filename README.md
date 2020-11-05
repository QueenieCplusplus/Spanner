# Spanner
GCP HA storage solution for DB migration

# Core Steps

(1) to set up Spanner instance (in GCP storage catalog)

(2) to create DB (schema & table)

(3) to commit column due to timestamp

(4) to load data

(5) to query from Sapnner DB

(6) to delete Spanner instance to free up resources

# Key Characters

1. HA

2. Scalability

3. Distribution

4. Relational DB

5. ACID (supported)

-----------------------------------------------

from step 1:

> to create a Spanner Instance

* 1.1, In Cloud Console, Nagivation Bar >> Storage >> Spanner 

![spanner](https://cdn.qwiklabs.com/oMyUIWRK4c6QIJwGdsCG40%2Fl6uaAZ8AsOAJ6eEvYTnA%3D)

form step 2:

> to create DB 

* 2.1, use Sample Code Below =>

    using System;
    using System.Threading.Tasks;
    using Google.Cloud.Spanner.Data;
    using CommandLine;

    namespace GoogleGame.ScoreBoard
    {
        [Verb("create", HelpText = "Create a sample Cloud Spanner database "
            + "along with sample 'Players' and 'Scores' tables in your project.")]
            
        class CreateOptions
        {
            [Value(0, HelpText = "The project ID of the project to use "
                + "when creating Cloud Spanner resources.", Required = true)]
            public string projectId { get; set; }
            [Value(1, HelpText = "The ID of the instance where the sample database "
                + "will be created.", Required = true)]
            public string instanceId { get; set; }
            [Value(2, HelpText = "The ID of the sample database to create.",
                Required = true)]
            public string databaseId { get; set; }
        }

        public class Program
        {
            enum ExitCode : int
            {
                Success = 0,
                InvalidParameter = 1,
            }

            public static object Create(string projectId,
                string instanceId, string databaseId)
            {
                var response =
                    CreateAsync(projectId, instanceId, databaseId);
                Console.WriteLine("Waiting for operation to complete...");
                response.Wait();
                Console.WriteLine($"Operation status: {response.Status}");
                Console.WriteLine($"Created sample database {databaseId} on "
                    + $"instance {instanceId}");
                return ExitCode.Success;
            }

            public static async Task CreateAsync(
                string projectId, string instanceId, string databaseId)
            {
                // Initialize request connection string for database creation.
                string connectionString =
                    $"Data Source=projects/{projectId}/instances/{instanceId}";
                using (var connection = new SpannerConnection(connectionString))
                {
                    string createStatement = $"CREATE DATABASE `{databaseId}`";
                    string[] createTableStatements = new string[] {
                      // Define create table statement for Players table.
                      @"CREATE TABLE Players(
                        PlayerId INT64 NOT NULL,
                        PlayerName STRING(2048) NOT NULL
                      ) PRIMARY KEY(PlayerId)",
                      // Define create table statement for Scores table.
                      @"CREATE TABLE Scores(
                        PlayerId INT64 NOT NULL,
                        Score INT64 NOT NULL,
                        Timestamp TIMESTAMP NOT NULL OPTIONS(allow_commit_timestamp=true)
                      ) PRIMARY KEY(PlayerId, Timestamp),
                          INTERLEAVE IN PARENT Players ON DELETE NO ACTION" };
                    // Make the request.
                    var cmd = connection.CreateDdlCommand(
                        createStatement, createTableStatements);
                    try
                    {
                        await cmd.ExecuteNonQueryAsync();
                    }
                    catch (SpannerException e) when
                        (e.ErrorCode == ErrorCode.AlreadyExists)
                    {
                        // OK.
                    }
                }
            }

            public static int Main(string[] args)
            {
                var verbMap = new VerbMap<object>();
                verbMap
                    .Add((CreateOptions opts) => Create(
                        opts.projectId, opts.instanceId, opts.databaseId))
                    .NotParsedFunc = (err) => 1;
                return (int)verbMap.Run(args);
            }
        }
    }

* 2.2, to ues Sample code below for project file which is called ScoreBoard.csproj=>


    <Project Sdk="Microsoft.NET.Sdk">

      <PropertyGroup>
        <OutputType>Exe</OutputType>
        <TargetFramework>netcoreapp2.1</TargetFramework>
      </PropertyGroup>

      <ItemGroup>
        <PackageReference Include="CommandLineParser" Version="2.8.0" />
        <PackageReference Include="Google.Cloud.Spanner.Data" Version="2.0.0" />
        <PackageReference Include="Google.Cloud.Spanner.V1" Version="2.0.0" />
      </ItemGroup>

      <ItemGroup>
        <ProjectReference Include="..\..\..\commandlineutil\Lib\CommandLineUtil.csproj" />
      </ItemGroup>

    </Project>

* 2.3, check Ref in the step 2.2 =>


    This change adds references:

    to the two C# Spanner Nuget packages "Google.Cloud.Spanner.Data" and "Google.Cloud.Spanner.V1" that you need to interact with the Cloud Spanner API.
    
    to the open source "CommandLineParser Nuget package" which is a handy library for handling command line input for console applications.
    
    to the "CommandLineUtil" project which is part of the dotnet-doc-samples Github repository and provides a useful "verbmap" extension to the CommandLineParser.
    
* 2.4, run the C# App =>
 
      $dotnet run
      
      $dotnet run create
      
      $dotnet run create [PROJECT_ID] [spanner instance name] [app file name]
      
      $dotnet run create [PROJECT_ID] [spanner name see step 1.1] scoreboard
      
      [output]
 
        Waiting for operation to complete...
        Operation status: RanToCompletion
        Created sample database scoreboard on instance cloudspanner-leaderboard

* 2.5, check resulting output in Cloud Console =>
 
  ![spanner db](https://cdn.qwiklabs.com/GT4%2BCTBJZHw3ADZ8SPHqrDXrYr5HkWkmJ7sgsPQkJqs%3D)

from step 3

> to add ACID feature code block and load data.

* 3.1, add insert cmd code block under "Verbmap"

    [Verb("insert", HelpText = "Insert sample 'players' records or 'scores' records "
            + "into the database.")]
        class InsertOptions
        {
            [Value(0, HelpText = "The project ID of the project to use "
                + "when managing Cloud Spanner resources.", Required = true)]
            public string projectId { get; set; }
            [Value(1, HelpText = "The ID of the instance where the sample database resides.",
                Required = true)]
            public string instanceId { get; set; }
            [Value(2, HelpText = "The ID of the database where the sample database resides.",
                Required = true)]
            public string databaseId { get; set; }
            [Value(3, HelpText = "The type of insert to perform, 'players' or 'scores'.",
                Required = true)]
            public string insertType { get; set; }
        }

* 3.2, add methods below beneath the existing method called "CreateAsync" =>

        public static object Insert(string projectId,
            string instanceId, string databaseId, string insertType)
        {
            if (insertType.ToLower() == "players")
            {
                var responseTask =
                    InsertPlayersAsync(projectId, instanceId, databaseId);
                Console.WriteLine("Waiting for insert players operation to complete...");
                responseTask.Wait();
                Console.WriteLine($"Operation status: {responseTask.Status}");
            }
            else if (insertType.ToLower() == "scores")
            {
                var responseTask =
                    InsertScoresAsync(projectId, instanceId, databaseId);
                Console.WriteLine("Waiting for insert scores operation to complete...");
                responseTask.Wait();
                Console.WriteLine($"Operation status: {responseTask.Status}");
            }
            else
            {
                Console.WriteLine("Invalid value for 'type of insert'. "
                    + "Specify 'players' or 'scores'.");
                return ExitCode.InvalidParameter;
            }
            Console.WriteLine($"Inserted {insertType} into sample database "
                + $"{databaseId} on instance {instanceId}");
            return ExitCode.Success;
        }

        public static async Task InsertPlayersAsync(string projectId,
            string instanceId, string databaseId)
        {
            string connectionString =
                $"Data Source=projects/{projectId}/instances/{instanceId}"
                + $"/databases/{databaseId}";
            Int64 numberOfPlayers = 0;
            using (var connection = new SpannerConnection(connectionString))
            {
                await connection.OpenAsync();
                using (var tx = await connection.BeginTransactionAsync())
                {
                    // Execute a SQL statement to get current number of records
                    // in the Players table.
                    var cmd = connection.CreateSelectCommand(
                        @"SELECT Count(PlayerId) as PlayerCount FROM Players");
                    cmd.Transaction = tx;
                    using (var reader = await cmd.ExecuteReaderAsync())
                    {
                        while (await reader.ReadAsync())
                        {
                            long parsedValue;
                            if (reader["PlayerCount"] != DBNull.Value)
                            {
                                bool result = Int64.TryParse(
                                    reader.GetFieldValue<string>("PlayerCount"),
                                        out parsedValue);
                                if (result)
                                {
                                    numberOfPlayers = parsedValue;
                                }
                            }
                        }
                    }
                    // Insert 100 player records into the Players table.
                    using (cmd = connection.CreateInsertCommand(
                        "Players", new SpannerParameterCollection
                    {
                        { "PlayerId", SpannerDbType.String },
                        { "PlayerName", SpannerDbType.String }
                    }))
                    {
                        cmd.Transaction = tx;
                        for (var x = 1; x <= 100; x++)
                        {
                            numberOfPlayers++;
                            cmd.Parameters["PlayerId"].Value =
                                Math.Abs(Guid.NewGuid().GetHashCode());
                            cmd.Parameters["PlayerName"].Value =
                                $"Player {numberOfPlayers}";
                            cmd.ExecuteNonQuery();
                        }
                    }
                    await tx.CommitAsync();
                }
            }
            Console.WriteLine("Done inserting player records...");
        }

        public static async Task InsertScoresAsync(
            string projectId, string instanceId, string databaseId)
        {
            string connectionString =
            $"Data Source=projects/{projectId}/instances/{instanceId}"
            + $"/databases/{databaseId}";

            // Insert 4 score records into the Scores table for each player in the Players table.
            using (var connection = new SpannerConnection(connectionString))
            {
                await connection.OpenAsync();
                Random r = new Random();
                bool playerRecordsFound = false;
                var cmdLookup = connection.CreateSelectCommand("SELECT * FROM Players");
                using (var reader = await cmdLookup.ExecuteReaderAsync())
                {
                    while (await reader.ReadAsync())
                    {
                        if (!playerRecordsFound)
                        {
                            playerRecordsFound = true;
                        }
                        using (var tx = await connection.BeginTransactionAsync())
                        using (var cmd = connection.CreateInsertCommand(
                            "Scores", new SpannerParameterCollection
                        {
                            { "PlayerId", SpannerDbType.String },
                            { "Score", SpannerDbType.Int64 },
                            { "Timestamp", SpannerDbType.Timestamp }
                        }))
                        {
                            cmd.Transaction = tx;
                            for (var x = 1; x <= 4; x++)
                            {
                                DateTime randomTimestamp = DateTime.Now
                                    .AddYears(r.Next(-2, 1))
                                    .AddMonths(r.Next(-12, 1))
                                    .AddDays(r.Next(-10, 1))
                                    .AddSeconds(r.Next(-60, 0))
                                    .AddMilliseconds(r.Next(-100000, 0));
                                cmd.Parameters["PlayerId"].Value =
                                    reader.GetFieldValue<int>("PlayerId");
                                // Insert random score value between 10000 and 1000000.
                                cmd.Parameters["Score"].Value = r.Next(1000, 1000001);
                                // Insert random past timestamp
                                // value into Timestamp column.
                                cmd.Parameters["Timestamp"].Value =
                                    randomTimestamp.ToString("o");
                                cmd.ExecuteNonQuery();
                            }
                            await tx.CommitAsync();
                        }
                    }
                    if (!playerRecordsFound)
                    {
                        Console.WriteLine("Parameter 'scores' is invalid since "
                        + "no player records currently exist. First insert players "
                        + "then insert scores.");
                        Environment.Exit((int)ExitCode.InvalidParameter);
                    }
                    else
                    {
                        Console.WriteLine("Done inserting score records...");
                    }
                }
            }
        }
