Setup
Pre-requisites
git
Github account
Terraform
AWS account
AWS CLI installed and configured
Docker with at least 4GB of RAM and Docker Compose v1.27.0 or later
Read this post , for information on setting up CI/CD, DB migrations, IAC(terraform), “make” commands and automated testing.

Run these commands to setup your project locally and on the cloud.

# Clone the code as shown below.
git clone https://github.com/josephmachado/bitcoinMonitor.git
cd bitcoinMonitor

# Local run & test
make up # start the docker containers on your computer & runs migrations under ./migrations
make ci # Runs auto formatting, lint checks, & all the test files under ./tests

# Create AWS services with Terraform
make tf-init # Only needed on your first terraform run (or if you add new providers)
make infra-up # type in yes after verifying the changes TF will make

# Wait until the EC2 instance is initialized, you can check this via your AWS UI
# See "Status Check" on the EC2 console, it should be "2/2 checks passed" before proceeding

make cloud-metabase # this command will forward Metabase port from EC2 to your machine and opens it in the browser
You can connect metabase to the warehouse with the configs in the env file. Refer to this doc for creating a Metabase dashboard.

Create database migrations as shown below.

make db-migration # enter a description, e.g., create some schema
# make your changes to the newly created file under ./migrations
make warehouse-migration # to run the new migration on your warehouse
For the continuous delivery to work, set up the infrastructure with terraform, & defined the following repository secrets. You can set up the repository secrets by going to Settings > Secrets > Actions > New repository secret.

SERVER_SSH_KEY: We can get this by running terraform -chdir=./terraform output -raw private_key in the project directory and paste the entire content in a new Action secret called SERVER_SSH_KEY.
REMOTE_HOST: Get this by running terraform -chdir=./terraform output -raw ec2_public_dns in the project directory.
REMOTE_USER: The value for this is ubuntu.
Project
For our project, we will pull bitcoin exchange data from CoinCap API . We will pull this data every 5 minutes and load it into our warehouse.

Architecture

1. ETL Code
The code to pull data from CoinCap API and load it into our warehouse is at exchange_data_etl.py . In this script we

Pull data from CoinCap API using the get_exchange_data function.
Use get_utc_from_unix_time function to get UTC based date time from unix time(in ms).
Load data into our warehouse using the _get_exchange_insert_query insert query.
def run() -> None:
    data = get_exchange_data()
    for d in data:
        d['update_dt'] = get_utc_from_unix_time(d.get('updated'))
    with WarehouseConnection(**get_warehouse_creds()).managed_cursor() as curr:
        p.execute_batch(curr, _get_exchange_insert_query(), data)
Ref: API data pull best practices

There are a few things going on at “with WarehouseConnection(**get_warehouse_creds()).managed_cursor() as curr:".

We use the get_warehouse_creds utility function to get the warehouse connection credentials.
The warehouse connection credentials are stored as environment variables within our docker compose definition. The docker-compose uses the hardcoded values from the env file.
The credentials are passed as **kwargs to the WarehouseConnection class.
The WarehouseConnection class uses contextmanager to enable opening and closing the DB connections easier. This lets us access the DB connection without having to write boilerplate code.
def get_warehouse_creds() -> Dict[str, Optional[Union[str, int]]]:
    return {
        'user': os.getenv('WAREHOUSE_USER'),
        'password': os.getenv('WAREHOUSE_PASSWORD'),
        'db': os.getenv('WAREHOUSE_DB'),
        'host': os.getenv('WAREHOUSE_HOST'),
        'port': int(os.getenv('WAREHOUSE_PORT', 5432)),
    }
class WarehouseConnection:
    def __init__(
        self, db: str, user: str, password: str, host: str, port: int
    ):
        self.conn_url = f'postgresql://{user}:{password}@{host}:{port}/{db}'

    @contextmanager
    def managed_cursor(self, cursor_factory=None):
        self.conn = psycopg2.connect(self.conn_url)
        self.conn.autocommit = True
        self.curr = self.conn.cursor(cursor_factory=cursor_factory)
        try:
            yield self.curr
        finally:
            self.curr.close()
            self.conn.close()
2. Test
Tests are crucial if you want to be confident about refactoring code, adding new features, and code correctness. In this example, we will add 2 major types of tests.

Unit test: To test if individual functions are working as expected. We test get_utc_from_unix_time with the test_get_utc_from_unix_time function.
Integration test: To test if multiple systems work together as expected.
For the integration test we

Mock the Coinbase API call using the mocker functionality of the pytest-mock library. We use fixture data at test/fixtures/sample_raw_exchange_data.csv as a result of an API call. This is to enable deterministic testing.
Assert that the data we store in the warehouse is the same as we expected.
Finally the teardown_method truncates the local warehouse table. This is automatically called by pytest after the test_covid_stats_etl_run test function is run.
class TestBitcoinMonitor:
    def teardown_method(self, test_covid_stats_etl_run):
        with WarehouseConnection(
            **get_warehouse_creds()
        ).managed_cursor() as curr:
            curr.execute("TRUNCATE TABLE bitcoin.exchange;")

    def get_exchange_data(self):
        with WarehouseConnection(**get_warehouse_creds()).managed_cursor(
            cursor_factory=psycopg2.extras.DictCursor
        ) as curr:
            curr.execute(
                '''SELECT id,
                        name,
                        rank,
                        percenttotalvolume,
                        volumeusd,
                        tradingpairs,
                        socket,
                        exchangeurl,
                        updated_unix_millis,
                        updated_utc
                        FROM bitcoin.exchange;'''
            )
            table_data = [dict(r) for r in curr.fetchall()]
        return table_data

    def test_covid_stats_etl_run(self, mocker):
        mocker.patch(
            'bitcoinmonitor.exchange_data_etl.get_exchange_data',
            return_value=[
                r
                for r in csv.DictReader(
                    open('test/fixtures/sample_raw_exchange_data.csv')
                )
            ],
        )
        run()
        expected_result = [
          {"see github repo for full data"}
        ]
        result = self.get_exchange_data()
        assert expected_result == result

See How to add tests to your data pipeline article to add more tests to this pipeline. You can run tests using

make up # to start all your containers 
make pytest
3. Scheduler
Now that we have the ETL script and tests setup. We need to schedule the ETL script to run every 5 minutes. Since this is a simple script we will go with cron instead of setting up a framework like Airflow or Dagster. The cron job is defined at scheduler/pull_bitcoin_exchange_info

SHELL=/bin/bash
HOME=/
*/5 * * * * WAREHOUSE_USER=sdeuser WAREHOUSE_PASSWORD=sdepassword1234 WAREHOUSE_DB=finance WAREHOUSE_HOST=warehouse WAREHOUSE_PORT=5432  PYTHONPATH=/code/src /usr/local/bin/python /code/src/bitcoinmonitor/exchange_data_etl.py


This file is placed inside the pipelinerunner docker container’s crontab location. You may notice that we have hardcoded the environment variables. Not having the environment variables hardcoded in this file is part of future work .

4. Presentation
Now that we have the code and scheduler set up, we can add checks and formatting automation to ensure that we follow best practices. This is what a hiring manager will be exposed to, when they look at your code. Ensuring that the presentation is clear, concise, and consistent is crucial.

4.1. Formatting, Linting, and Type checks
Formatting enables us to stay consistent with the code format. We use black and isort to automate formatting. The -S black module flag ensures that we use single quotes for strings (following PEP8).

Linting analyzes the code for potential errors and ensures that the code formatting is consistent. We use flake8 to lint check our code.

Type checking enables us to catch type errors (when defined). We use mypy for this.

All of these are run within the docker container. We use a Makefile to store shortcuts to run these commands.

4.2. Architecture Diagram
Instead of having a long text, it is usually easier to understand the data flow with an architecture diagram. It does not have to be beautiful, but must be clear and understandable. Our architecture diagram is shown below.

Architecture

4.3. README.md
The readme should be clear and concise. It’s a good idea to have sections for

Description of the problem
Architecture diagram
Setup instructions
You can automatically format and test your code with

make ci # this command will format your code, run lint and type checks and run all your tests 
After which, you can push it to your Github repository.

5. Adding Dashboard to your Profile
Refer to Metabase documentation on how to create a dashboard . Once you create a dashboard, get its public link following the steps here . Create a hyperlink to this dashboard from your resume or LinkedIn page. You can also embed the dashboard as an iframe on any website.

A sample dashboard using bitcoin exchange data is shown below.

Dash

Depending on the EC2 instance type you choose you may occur some cost. Use AWS cost calculator to figure out the cost.

Future Work
Although this provides a good starting point, there is a lot of work to be done. Some future work may include

Data quality testing
Better scheduler and workflow manager to handle backfills, reruns, and parallelism
Better failure handling
Streaming data from APIs vs mini-batches
Add system env variable to crontab
Data cleanup job to remove old data, since our Postgres is running on a small EC2 instance
API rate limiting
Tear down infra
After you are done, make sure to destroy your cloud infrastructure.

make down # Stop docker containers on your computer
make infra-down # type in yes after verifying the changes TF will make
Conclusion
Building data projects are hard. Getting hiring managers to read through your Github code is even harder. By focusing on the right things, you can achieve your objective. In this case, the objective is to show your data skills to a hiring manager. We do this by making it extremely easy for the hiring manager to see the end product, code, and architecture.

Hope this article gave you a good idea of how to design a data project to impress a hiring manager. If you have any questions or comments please leave them in the comment section below.

Further Reading
Airflow scheduling
Beginner DE project: batch
Adding data tests
API data pull using lambda
dbt, getting started
References
AWS EC2 connection issues
Ubuntu docker install
Crontab env variables
Metabase documentation
Coincap API
