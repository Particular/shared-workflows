# The NuGet Audit shared workflow

The purpose of this workflow is to run a `dotnet restore` and react when the restore reports any CVEs in our NuGet dependencies.

This shared workflow is called via [a simple workflow](https://github.com/Particular/RepoStandards/blob/master/.github/workflows/nuget-audit.yml) deployed to each repository that uses [RepoStandards Synchronization](https://github.com/Particular/Platform/blob/main/guidelines/repository-standards.md#repostandards-synchronization).

Internal Automation runs this workflow daily via [the Weekly CI scheduled function](https://github.com/Particular/InternalAutomation/blob/master/docs/functions.md#run-weekly-ci)

## Key technical details

### Secrets

The shared workflow [inherits secrets](https://docs.github.com/en/actions/sharing-automations/reusing-workflows#using-inputs-and-secrets-in-a-reusable-workflow) from the calling workflow distributed via [RepositoryStandards](https://github.com/Particular/RepoStandards).

### CVE detection mechanism

The workflow detects CVEs via [the package auditing feature of `dotnet restore`](https://learn.microsoft.com/en-us/nuget/concepts/auditing-packages).

The workflow detects if `dotnet restore` reports any of the following warning codes:

- NU1901
- NU1902
- NU1903
- NU1904

If so it will exit with a number equal to the number of warning codes detected (total, not distinct), causing it to fail the workflow if any are detected.

### Restore data file

If the workflow has failed because [CVEs have been detected](#cve-detection-mechanism), the workflow gathers information for further processing by [Internal Automation](https://github.com/Particular/InternalAutomation):

- The GitHub repository ID that called this shared workflow
- The name of the repository branch that called this shared workflow
- The current commit signature of the branch
- A json file that combines via the [jq command line tool](https://jqlang.org/):
   - The output of the `dotnet list package --vulnerable --include-transitive` command output as json
   - The json content of all of the project.asset.json files created by running `dotnet restore`

The workflow uploads this data via a curl command to the `ProcessNuGetAuditResults` HTTP function in Internal Automation using [an Azure Functions host access key](https://learn.microsoft.com/en-us/azure/azure-functions/function-keys-how-to?tabs=azure-portal).

### Archives

The workflow archives the files created as well as a log file that outputs the results of the `dotnet restore`.