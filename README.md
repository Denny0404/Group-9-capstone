# Complete Project

This repository contains a simple web application along with
infrastructure as code and CI/CD configuration.  The goal of the
project is to demonstrate end‑to‑end DevOps practices, from
application development through automated testing, code review and
deployment to AWS.

## Contents

* **backend/** – A minimal REST API built with Python’s built‑in
  modules.  The server provides CRUD operations for a `users`
  resource and persists data in an SQLite database.  It does not
  rely on any external dependencies which makes it suitable for
  restricted environments.
* **frontend/** – A static HTML page that interacts with the API via
  the Fetch API.  It allows you to list users, create new users
  and delete existing ones.
* **infrastructure/** – Modular AWS CloudFormation templates that
  provision a VPC, public subnet, security group and EC2 instance.
  The templates are split into `network.yaml`, `compute.yaml` and
  `root.yaml` following recommended best practices for nested
  stacks【968227933497382†L245-L259】.
* **tests/** – Unit tests written with Python’s `unittest` module.
  These tests exercise the database helper functions to ensure that
  CRUD operations behave correctly.
* **.github/workflows/** – A GitHub Actions workflow that runs tests,
  performs CodeQL analysis and deploys the application via
  CloudFormation.  The pipeline is triggered on pushes to `main`,
  `develop` and `feature/*` branches and on pull requests to `main`.

## Backend API

The API is implemented in [`backend/server.py`](backend/server.py).  It
uses Python’s `http.server` to handle requests and the `sqlite3`
module to persist data.  The API exposes the following endpoints:

| Method | Path            | Description                    |
|-------:|:---------------|:-------------------------------|
| `GET`  | `/users`       | Return a JSON list of all users |
| `GET`  | `/users/{id}`  | Return a single user by ID      |
| `POST` | `/users`       | Create a new user               |
| `PUT`  | `/users/{id}`  | Update an existing user         |
| `DELETE` | `/users/{id}` | Delete a user                   |

To run the API locally:

```bash
cd complete_project/backend
python server.py
# The server will start on http://localhost:8000
```

### Database Utilities

The `backend/db_utils.py` module exposes helper functions `init_db()`,
`get_all_users()`, `get_user()`, `create_user()`, `update_user()` and
`delete_user()`.  These functions are used by the HTTP handler and are
also imported directly in the tests.

## Front‑end

The front‑end is a single HTML file located at
[`frontend/index.html`](frontend/index.html).  It contains a simple
table to display users and a form for creating new users.  The page
uses JavaScript’s Fetch API to call the backend.  To use it locally,
open the file in a browser **after** starting the API.  The front‑end
assumes the API is reachable at `http://localhost:8000`.

## Infrastructure as Code

The `infrastructure` folder contains three CloudFormation templates:

1. **network.yaml** – Defines a VPC, an Internet Gateway, a public
   subnet and associated route tables.  Outputs export the VPC and
   subnet IDs so they can be imported by other stacks.
2. **compute.yaml** – Creates a security group and a single EC2
   instance.  The instance’s user data installs Python, downloads the
   application artefact from S3 and starts the server.  Ports 22 and
   8000 are opened for SSH and application access respectively.
3. **root.yaml** – Nests the network and compute stacks.  It passes
   parameters through to the nested templates and outputs the public
   URL of the instance.  Using nested stacks promotes modularity and
   reuse【968227933497382†L245-L259】.

Before deploying the infrastructure you must upload the nested
templates (`network.yaml` and `compute.yaml`) to an S3 bucket and set
the `NetworkTemplateUrl` and `ComputeTemplateUrl` parameters in
`root.yaml` accordingly.  You must also publish a zipped version of
this project to the same bucket (see the deploy step in the CI/CD
workflow).

## CI/CD Pipeline

The GitHub Actions workflow defined in
[`.github/workflows/ci-cd.yml`](.github/workflows/ci-cd.yml) implements
a multi‑stage pipeline:

1. **Source** – The pipeline triggers on pushes and pull requests.
   The branching strategy encourages keeping a `main` branch for
   production and working on feature branches.  Pull requests to `main`
   require review before merging【651065683961189†L74-L159】.
2. **Test** – Python unit tests are executed via
   `unittest discover`.  The project does not require third‑party
   dependencies, so no `pip install` is needed.
3. **Code scanning** – The workflow initializes CodeQL for Python and
   performs a static analysis pass.  CodeQL treats code as data and
   helps discover common security vulnerabilities【462449089142040†L549-L584】.
4. **Deploy** – On pushes to `main`, the pipeline packages the
   CloudFormation templates, uploads them to S3 and deploys the root
   stack using the AWS CLI.  The deployment step uses AWS secrets
   configured in the repository.

To enable the deployment step in your own environment you must
configure the following secrets in GitHub:

| Secret name       | Description                                    |
|------------------|--------------------------------------------------|
| `AWS_ACCESS_KEY_ID`     | Access key for an IAM user with CloudFormation permissions |
| `AWS_SECRET_ACCESS_KEY` | Secret access key for the IAM user          |
| `CF_BUCKET`             | S3 bucket to store CloudFormation templates |
| `ARTIFACT_BUCKET`       | S3 bucket containing the zipped application  |
| `ARTIFACT_KEY`          | Key in the artifact bucket (e.g. `app.zip`)  |
| `EC2_KEY_NAME`          | Name of an EC2 KeyPair for SSH access        |

The workflow uses branch protection rules and pull requests to enforce
reviews and passing builds before code is merged.  Microsoft’s
guidance recommends keeping the branch strategy simple and using
feature branches with pull requests【651065683961189†L74-L159】.

## Testing

Run tests locally using the following command:

```bash
cd complete_project
python -m unittest discover -s tests -p 'test_*.py' -v
```

Each test case creates a temporary SQLite database to ensure isolation.
The tests cover all CRUD operations for users.  Although this project
does not use Jest (a JavaScript test runner), the concept of
enforcing coverage thresholds is illustrated in the Jest documentation,
which shows how to fail builds when coverage drops below 80 %【379955823890011†L506-L533】.  You
could adopt a similar strategy using Python’s `coverage.py` if desired.

## Monitoring and Logging

While the included templates deploy a basic EC2 instance, you should
instrument your application with structured JSON logs and forward
them to CloudWatch.  The AWS best‑practices guide recommends
structuring logs and using the CloudWatch agent or the `awslogs`
driver to centralize them【826228774271274†L166-L183】.  Retention
policies should be configured to control storage costs【826228774271274†L197-L202】.

## Development Guidelines

* Use feature branches for new work and create pull requests into
  `main`.  Keep branch names consistent and descriptive.
* Require at least one reviewer to approve a pull request before
  merging.  A passing CI build should be mandatory【651065683961189†L147-L159】.
* Keep commit messages clear and concise.  Each team member should
  contribute at least five commits to the project.
* Document any infrastructure changes thoroughly and parameterize
  templates to make them reusable across environments.

## Running the Project Locally

1. **Backend:** Start the API by running `python server.py` in the
   `backend` directory.  It listens on port 8000 by default.
2. **Front‑end:** Open `frontend/index.html` in a browser.  Use the
   form to create users and the refresh button to view existing users.
3. **Tests:** Execute the unit tests with `python -m unittest` as
   described above.
4. **Infrastructure:** To deploy to AWS manually, first zip the
   repository and upload it along with the nested templates to an S3
   bucket.  Then run `aws cloudformation deploy` with the appropriate
   parameters or rely on the GitHub Actions workflow.

## Licence

This project is provided for educational purposes.  You are free to
modify and reuse it as needed.