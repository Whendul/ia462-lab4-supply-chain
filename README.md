IA462 Lab 4: Supply Chain Security
This repository contains a small Flask application that runs in a Docker container. I used it to practice managing dependencies, checking for known vulnerabilities, and keeping track of the software included in an image. The container runs as 'appuser' instead of root to limit the permissions available to the application.
Pinning vs. Floating Dependencies
The application uses Flask version 3.1.3 and Werkzeug version 3.1.9, which are listed in 'requirements.txt'. These exact versions help prevent an installation from unexpectedly switching to a different release. The 'requirements.lock' file also records the resolved dependencies and their hashes, and the Docker build requires downloaded packages to match those hashes.

The Docker base image is pinned to a SHA256 digest. This identifies the specific base image instead of depending only on a tag that could change. The Dockerfile also installs available Debian package updates, so later builds can include newer operating system packages. Pinning improves consistency, but the entire build is not guaranteed to be identical every time.

GitHub Dependency Security Tools
I enabled the dependency graph, Dependabot alerts, and Dependabot security updates. These features help show which dependencies the project uses and identify packages with known vulnerabilities. The configuration in '.github/dependabot.yml' checks weekly for updates to Python packages, Docker images, and GitHub Actions.

I also enabled secret scanning and push protection. These features help detect supported types of exposed credentials and block pushes that contain recognized secrets. They reduce the chance of accidentally publishing sensitive information in the repository.

I enabled branch protection for 'main'. The rule requires a pull request, one approving review from someone with write access, and all five status checks to pass. In the vulnerable dependency test, the checks were marked as required and GitHub blocked the merge.
Dependency Confusion & Supply Chain Attacks
Dependency confusion can happen when a package manager installs an unexpected public package instead of the intended internal package. This shows why checking a package name, source, and publisher matters before adding it to a project.

In this lab, exact versions and installation hashes help reduce unexpected changes to the dependencies. The hashes verify that downloaded files match the files recorded in the lock file. They do not prove that a package is trustworthy, so dependency review and vulnerability scanning are also part of the setup.

Securing Dependencies in GitHub Actions
The workflows reference actions using full commit SHAs. This keeps each action tied to a particular revision and reduces the risk of a moving tag changing the code used by the workflow.

The workflows use 'contents: read' permissions, and checkout steps do not keep Git credentials available for later steps. These settings limit the access available during workflow execution. The Docker image is built once in the pipeline and shared with the image scanning and SBOM jobs so those jobs examine the same image.

Software Bill of Materials (SBOM)
I used Syft to generate an inventory of the software in the Docker image. The outputs include SPDX JSON, CycloneDX JSON, and a readable table. The local files are stored in 'sboms/', along with the image ID.

The local SPDX export contains 97 package entries. I added the snapshots through pull request 3, which passed all five checks before being merged. The pipeline also produces SBOM files as downloadable artifacts. These inventories help identify installed components when investigating a vulnerability or comparing builds.

I also exported an SBOM from GitHub through Insights and the dependency graph. The downloaded JSON uses SPDX version 2.3 and lists repository dependencies, including Python packages and GitHub Actions. The Syft exports describe the software in the Docker image, so the two sources cover different parts of the project.

Integrating Dependency Scanning into CI/CD
The security pipeline builds the Docker image, audits Python dependencies, scans the image with Trivy, and generates an SBOM. It runs for changes pushed to 'main' and for pull requests targeting 'main'.

The Python dependency audit checks both the direct requirements and the hashed lock file for known vulnerabilities. Trivy creates a report covering all configured severity levels, including findings without available fixes. A separate Trivy check fails when it finds a HIGH or CRITICAL vulnerability with an available fix.

The workflow saves the scan and audit reports as artifacts, including when a security check fails. This makes it possible to inspect the reason for a failure. Passing these checks means the build meets the configured scan rules at that time, rather than proving that the software has no vulnerabilities.

GitHub's Dependency Review Action
Dependency Review examines dependency changes introduced by a pull request. The workflow is configured to fail when a change introduces a known vulnerability rated HIGH or CRITICAL. It also checks licenses against the configured denied GPL version 2 and AGPL version 3 identifiers.

This adds another opportunity to catch risky dependency changes before they reach 'main'. Dependency Review is a required check in the branch protection rule. In pull request 5, it detected vulnerable packages and failed after I changed Flask and Werkzeug to version 2.0.0.

How to reproduce the vulnerable PR demo
I completed the demonstration in pull request 5. Dependency Review, the Python dependency audit, and Trivy failed. The Docker build and SBOM generation passed. The Python audit reported known vulnerabilities in Flask and Werkzeug version 2.0.0. GitHub blocked the merge, and I closed the pull request without merging it. The versions on 'main' stayed at Flask 3.1.3 and Werkzeug 3.1.9, and the successful pipeline run on 'main' remained unchanged.

The following steps explain how to repeat the test.
1. Update the local 'main' branch and create a separate test branch.
2. Change 'requirements.txt' to use 'flask==2.0.0' and 'werkzeug==2.0.0'.
3. Regenerate 'requirements.lock' from the changed requirements with hash generation enabled.
4. Commit the updated requirements files and push the test branch.
5. Open a pull request targeting 'main' and wait for the checks to finish.
6. Inspect the Dependency Review and Python dependency audit results. These checks are expected to fail because the test introduces versions with known vulnerabilities. Inspect the Trivy results as well.
7. Save screenshots and reports showing the actual failures and the blocked merge.
8. Close the pull request without merging it, then return to 'main'.

The purpose of this test is to show that the security checks detect the vulnerable dependencies and that branch protection prevents the failing pull request from being merged.
