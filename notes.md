# Notes

## AI Disclosure
I used Claude (Anthropic) to assist with this assignment. Specifically:
- Helping me understand GitHub Actions concepts and syntax I wasn't familiar with
- Cleaning up these notes
- Helping me understand error messages and what they meant

---

## Bug Fixes (Task 3)

Bug 1: Bash assignment syntax
Looking at the error message in github actions, it said the command is not found, which made me think it was a syntax error. I then checked how to assign a variable in a bash script and found that spaces around the = sign are not valid — `appName = "Test App"` is interpreted as a command, not an assignment. The correct syntax is `appName="Test App"`.

Bug 2: variable interpolation syntax
Similar process — the original code used `${$appName}` which is invalid bash syntax. The double $ causes a "bad substitution" error. The correct syntax is `${appName}`.

Bug 3: added chmod +x to make scripts executable
pretty self explanatory, needed permissions to run the scripts. Without it the runner throws "Permission denied".

Bug 4: updated platform to match build.sh expected input
there was a missing capitalization — the workflow passed `android` but build.sh only accepts `Android` or `iOS` with a capital first letter.

Bug 5: correct upload.sh arguments and remove stray quote
the argument had a stray quote, and then I checked upload.sh for the proper syntax to call it. Saw that it requires 2 arguments: the platform and the artifact path.

Bug 6: add UPLOAD_TOKEN secret to upload step environment
after running the last build, the error stated that UPLOAD_TOKEN was missing from the environment, and from that I learned that 1. the script requires an environment variable called UPLOAD_TOKEN, and since it's clearly meant to be a credential. 2. It should be stored as a secret in GitHub and injected at runtime via an env: block.

Bug 7: 
The pipeline reported success but only uploaded one file (Test App.apk). Reading build.sh revealed a loop that runs 10 times generating 10 additional asset files (asset-1.dat through asset-10.dat), confirmed in the Run Build logs. The upload step was hardcoded to a single file path, so the 10 asset files were never uploaded.

To fix this, replaced the hardcoded upload path with a find command that discovers all files in the build directory and loops through them, calling upload.sh once per file.

Additionally, the upload step needed the app name to correctly locate the build directory, but appName was defined as a bash variable inside the Run Build step and is not accessible to other steps. Refactored to use $GITHUB_OUTPUT which persists values across steps for the entire job. Used AI to learn about $GITHUB_OUTPUT and how to implement it.


## Task 4

workflow_dispatch is a GitHub Actions specific keyword that tells the workflow what event should trigger it. It specifically means "this workflow can be triggered manually by a human clicking a button in the GitHub Actions UI". It also supports an inputs: section which adds form fields to the Run workflow dialog.

Used AI to read through the documentation on github and help understand the inputs syntax.
https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/events-that-trigger-workflows#workflow_dispatch

## Task 5

**CDN** stands for Content Delivery Network and is essentially a network of servers spread geographically that store and serve files.

git commit SHA is just a unique ID for a commit. Used as a fallback version string so every build is automatically versioned without the engineer having to supply one.

'-z' means "is this string empty?" (was used in upload.sh to check for the upload token, so borrowed from that)
'git rev-parse --short HEAD' is a git command that returns the short commit SHA of the current commit.

if: github.event_name == 'push' && github.ref == 'refs/heads/production'
used AI to find out how to access which event was running and how to check if the branch was production.

had to add a default fallback for the project name when the pipeline is triggered by a push, since there are no inputs on a push trigger.

## Task 6

build.sh already has a --profile flag built in.

so all I need to do is expose this flag:
- add a boolean input
- conditionally pass --profile flag when boolean is true
- save the .pro files as artifact

just added a new input to the on: block

then in the Run Build step I check the input and add the flag if it's true

then to store the .pro files I use actions/upload-artifact

'uses:' invokes a pre-built action written by GitHub. `actions/upload-artifact` is an official GitHub action that saves files to GitHub's storage so they can be downloaded from the Actions UI.

'with:' is how options are passed to a 'uses:' action (similar to how 'env:' passes environment variables)

## Task 7

Looking up how to run the same job multiple times with different parameters.

Tried using matrix strategy but didn't work — matrix.platform cannot be referenced in a job-level if: condition in GitHub Actions.

Matrix strategy - lets you define the job once and tell Actions to run the job for each item on the list. And it happens in parallel.

ended up creating two separate jobs instead, one for android and one for iOS, each with their own simple if: condition. The two jobs still run in parallel.

## Task 8

upgraded actions/checkout from v2 to v4 — v2 was generating a deprecation warning in every run. Note: v4 still shows a warning about Node.js 24 compatibility coming June 2026, which is outside our control until a newer version is released.

also saved build artifacts to GitHub Actions so engineers can download them directly from the UI without needing CDN access. Set to 30 day retention.
