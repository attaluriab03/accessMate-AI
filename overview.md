High-Level Workflow
- GitHub App that developers can add to their repo (public/private)
- Developer opens PR
- AccessMate downloads the files and runs a set of accessibility scans
    - based on the WCAG guidelines, it will detect issues and explain them to dev
- In the PR view, developer will then see options like
    - View Details --> shows explanation + code suggestion
    - Create Fix PR --> tells AccessMate to generate a patch branch
    - Ignore --> mark as known and ignore the suggested fix/issue
- If Create Fix PR is selected
    - creates a new branch
    - adds the code changes
    - runs a build to make sure nothing breaks
    - opens a new PR with the patch + accessibility test results
- For the accessibility test results, can use Lighthouse (google application)
    - runs a before/after test and attaches a small report to the PR

- Developer can then review the auto-fix PR and merge