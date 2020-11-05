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


-----------------------------------------------

from step 1:

> to create a Spanner Instance

1.1, In Cloud Console, Nagivation Bar >> Storage >> Spanner 

![spanner](https://cdn.qwiklabs.com/oMyUIWRK4c6QIJwGdsCG40%2Fl6uaAZ8AsOAJ6eEvYTnA%3D)

form step 2:

> to create DB 

2.1, use Sample Code Below =>

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

2.2, to ues Sample code below for project file which is called ScoreBoard.csproj=>


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

2.3, check Ref in the step 2.2 =>


    This change adds references:

    to the two C# Spanner Nuget packages "Google.Cloud.Spanner.Data" and "Google.Cloud.Spanner.V1" that you need to interact with the Cloud Spanner API.
    
    to the open source "CommandLineParser Nuget package" which is a handy library for handling command line input for console applications.
    
    to the "CommandLineUtil" project which is part of the dotnet-doc-samples Github repository and provides a useful "verbmap" extension to the CommandLineParser.
    
 2.4, run the C# App =>
 
      $dotnet run
      
      $dotnet run create
      
      $dotnet run create [PROJECT_ID] [spanner name see step 1.1] scoreboard
      
      [output]
 
     ScoreBoard 1.0.0
     Copyright (C) 2020 ScoreBoard

     ERROR(S):
      No verb selected.

      create     Create a sample Cloud Spanner database along with sample 'Players' and 'Scores' tables in your project.

      help       Display more information on a specific command.

      version    Display version information.

 2.5, check resulting output in Cloud Console =>
 
  ![spanner db](https://cdn.qwiklabs.com/GT4%2BCTBJZHw3ADZ8SPHqrDXrYr5HkWkmJ7sgsPQkJqs%3D)
