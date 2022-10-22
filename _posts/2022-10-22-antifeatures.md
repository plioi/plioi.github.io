---
title: 'Antifeatures'
layout: post
---

The primary design goal for the [Fixie test framework](https://github.com/fixie/fixie) is for its customization hooks to make new feature requests irrelevant. It should be easier to accomplish your opinionated goal in your code than to wait on some feature to be added that may or may not fit your needs exactly.

Consider GitHub Actions integration. GitHub added [Job Summaries](https://github.blog/2022-05-09-supercharging-github-actions-with-job-summaries/), a hook that lets your Step contribute arbitrary markdown content to the overall results page.

All you have to do is output markdown-formatted text to the file found by inspecting the `GITHUB_STEP_SUMMARY` environment variable. GitHub Actions then collects all of these for presentation. By constraining your contribution to markdown, you can say what you need to say without risk of breaking the design of the surrounding page.

Job Summaries are a great customization hook because they balance three concerns:

1. They are incredibly simple to use.
2. They give users a great deal of creative control.
3. They provide enough constraints to keep you from getting into trouble.

Fixie's own reporting hook provides a similar balance.

In the case of integrating with GitHub Actions, your own test projects can get opinionated about how much or how little to include in your Job Summary. In my case, I just wanted a simple color-coded summary of a test run, trusting the details would be captured in the Step log.

All it took was adding a simple event-based report class to [write the markdown the way I wanted](https://github.com/fixie/fixie/pull/293), and then to enlist that report when we're running inside the Continuous Integration environment:

```cs
class GitHubReport :
    IHandler<ExecutionStarted>,
    IHandler<ExecutionCompleted>
{
    readonly TestEnvironment environment;

    public GitHubReport(TestEnvironment environment)
        => this.environment = environment;

    public async Task Handle(ExecutionStarted message)
    {
        var assembly = Path.GetFileNameWithoutExtension(environment.Assembly.Location);
        var framework = environment.TargetFramework;

        await AppendToJobSummary($"## {assembly} ({framework}){NewLine}{NewLine}");
    }

    public async Task Handle(ExecutionCompleted message)
    {
        string severity;
        string detail;

        if (message.Total == 0)
        {
            severity = "red";
            detail = "No tests found.";
        }
        else
        {
            severity = "green";

            var parts = new List<string>();

            if (message.Passed > 0)
                parts.Add($"{message.Passed} passed");

            if (message.Skipped > 0)
            {
                severity = "yellow";
                parts.Add($"{message.Skipped} skipped");
            }

            if (message.Failed > 0)
            {
                severity = "red";
                parts.Add($"{message.Failed} failed");
            }

            detail = string.Join(", ", parts);
        }

        await AppendToJobSummary($":{severity}_circle: {detail}{NewLine}{NewLine}");
    }

    static async Task AppendToJobSummary(string summary)
    {
        if (GetEnvironmentVariable("GITHUB_STEP_SUMMARY") is string summaryFile)
            await File.AppendAllTextAsync(summaryFile, summary);
    }
}
```

```cs
class TestProject : ITestProject
{
    public void Configure(TestConfiguration configuration, TestEnvironment environment)
    {
        if (environment.IsContinuousIntegration())
            configuration.Reports.Add(new GitHubReport(environment));
    }
}
```

When my build fails, the first screen I arrive at gives me a quick attention-getting summary of the test run, so I know where to dig deeper next:

<img src="/images/2022/10/job-summary-test-failures.png" alt="GitHub Job Summary Indicating Test Failures" title="GitHub Job Summary" />

As another example, during development [I want my diff tool to open up upon failures involving large string comparisons](https://github.com/fixie/fixie/blob/0bcf9de5541507f141a3c8882ef591a5922c1c07/src/Fixie.Tests/DiffToolReport.cs), so long as there's only one such failure to present:

```cs
class DiffToolReport : IHandler<TestFailed>, IHandler<ExecutionCompleted>
{
    int failures;
    Exception? singleFailure;

    public Task Handle(TestFailed message)
    {
        failures++;

        singleFailure = failures == 1 ? message.Reason : null;

        return Task.CompletedTask;
    }

    public async Task Handle(ExecutionCompleted message)
    {
        if (singleFailure is AssertException exception)
            if (!exception.HasCompactRepresentations)
                await LaunchDiffTool(exception);
    }

    static async Task LaunchDiffTool(AssertException exception)
    {
        var tempPath = Path.GetTempPath();
        var expectedPath = Path.Combine(tempPath, "expected.txt");
        var actualPath = Path.Combine(tempPath, "actual.txt");

        File.WriteAllText(expectedPath, exception.Expected);
        File.WriteAllText(actualPath, exception.Actual);

        await DiffRunner.LaunchAsync(expectedPath, actualPath);
    }
}
```

Naturally we only want either report to run, one in CI builds and one during development:

```cs
    class TestProject : ITestProject
    {
        public void Configure(TestConfiguration configuration, TestEnvironment environment)
        {
            if (environment.IsDevelopment())
                configuration.Reports.Add<DiffToolReport>();
            else
                configuration.Reports.Add(new GitHubReport(environment));
        }
    }
```

If these were first-class features of the framework, I'd be chasing bugs and adjustments and varied opinions and "Yeah but what about my bizarre situation"s forever. By providing useful hooks, by providing an "antifeature", the *lack* of feature overload *is* the feature.