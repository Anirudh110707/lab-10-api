# Lab 10: Serve the Book Analysis as a Web API

> [!CAUTION]
> **Do NOT click Reset in the AWS Academy Learner Lab unless your instructor tells you to.**
> Reset permanently deletes everything in your lab account (S3 buckets, EC2 instances, Lambda functions, ...) and cannot be undone. To pause, click **End Lab** instead. Your resources will remain available the next time you click **Start Lab**.

**The Case:** A classmate wants to build a web page that shows your books with their summaries, and a research group wants to pull the word statistics into a notebook. Both need your book analysis: the word statistics from [Lab 07](https://github.com/ksiller/lab-07-ec2-lambda) and the summaries and genres from [Lab 08](https://github.com/ksiller/lab-08-hpc). The analysis is stored in your MongoDB Atlas collection `gutenberg.books`, which only you can read, and neither the classmate nor the research group should get your Atlas password. In this lab you give them a URL instead: a read-only **web API** that anyone, or any program, can call to get the books and their analysis as JSON. The API runs only when someone calls the URL, and costs nothing while no requests come in.

**Goal:** You extend your Lab 07 Chalice app `book-analyzer` with three web endpoints, test them on your computer, and deploy them to AWS: **API Gateway**, an AWS service that gives your API a public URL, receives each request and runs a Lambda function, which reads from MongoDB. In [Class 10](https://github.com/ksiller/DS2022/blob/main/class/10-fastapi/README.md) you built a Book API with FastAPI that ran on your computer, from a fixed dictionary in its code; this one runs in the cloud, on your real data.

> [!IMPORTANT]
> `mst3k` is a placeholder. Replace it everywhere with your UVA computing ID, for example `public-mst3k` → `public-abc2de`. If your Lab 06 bucket has a different name, use that name. Likewise, replace `YOUR_USERNAME` with your GitHub user name.

## Learning objectives

- Explain what a web API is: methods, endpoints (routes), path and query parameters, JSON responses, and status codes
- Build an API with several query endpoints using Chalice
- Test an API on your computer (`chalice local`) before you deploy it
- Deploy the API with `chalice deploy`, and know what it creates in AWS: an API in API Gateway and a second Lambda function
- Call an API from the command line with `curl`, and read its status code and headers
- Compare a serverless API (Chalice) with an API served by a running program (FastAPI, Class 10)


The lab builds on:

- [Lab 05: NoSQL with MongoDB](https://github.com/ksiller/lab-05-nosql): `find`, projections, and `count_documents`
- [Lab 07: EC2 and Lambda](https://github.com/ksiller/lab-07-ec2-lambda): your Chalice app `book-analyzer`, and the SSM parameter `mongodb_credentials`
- [Lab 08: HPC](https://github.com/ksiller/lab-08-hpc): the `llm_analysis` fields in your documents
- [Class 10: FastAPI](https://github.com/ksiller/DS2022/blob/main/class/10-fastapi/README.md): endpoints, path parameters, `curl`, status codes, and a Book API with FastAPI

## What is in this repository

```text
lab-10-api/
├── README.md
├── pyproject.toml              # the Python packages uv installs: boto3, pymongo, and chalice (for testing and deploying)
├── uv.lock
├── .python-version
├── src/book_analysis/
│   └── analyze.py              # the Lab 07 analysis, with the connect() function for Atlas (provided, do not change)
├── KNOWLEDGE_CHECK.md          # questions you answer for your submission
├── docs/                       # diagrams for this README
└── CLAUDE.md                   # instructions for Claude Code, if you use it
```

You add these yourself:

- `book-analyzer/`: your Chalice app from Lab 07, copied in Step 1.4, with three new routes
- `results/`, containing the outputs for your submission

Write your answers in `KNOWLEDGE_CHECK.md`. Do not modify `README.md`.

## APIs, endpoints, and requests

An **API** (application programming interface) is a way for programs to talk to each other. A **web API** does it over HTTP, like a web browser, but it answers with data, usually JSON, instead of web pages. Every request names a **method** and a URL:

```text
GET https://abc123.execute-api.us-east-1.amazonaws.com/api/books?author=Mark%20Twain
└┬┘ └───────────────────────┬────────────────────────┘└┬─┘└──┬─┘└────────┬─────────┘
method            server (API Gateway)               stage path   query parameter
```

When you deploy, API Gateway gives your API its server name; yours will have another code instead of `abc123`. `/api` is the **stage**: API Gateway can serve several versions of an API, and Chalice names your version `api`, so every path of your API will start with `/api`. It is like `/v1` in the Open-Meteo URL from class.

| Term | Meaning | In this lab |
| --- | --- | --- |
| **method** | what the request wants to do: `GET` fetches data, `POST` sends data | `GET` for every endpoint you build |
| **endpoint** (route) | a path the API answers, together with its method | `GET /`, `GET /books`, `GET /books/{key}` |
| **path parameter** | a part of the path that changes, written `{key}` in the route | `/books/pg11.txt`: `key` is `pg11.txt` |
| **query parameter** | `name=value` after `?`, separated by `&`, usually optional | `/books?author=Mark%20Twain`; `%20` is a space, encoded for the URL |
| **status code** | a three-digit number in the response that says how it went | `200` OK, `403` forbidden, `404` not found, `500` error in the server |

Status codes in the `4xx` range mean the **client** asked for something wrong (a book or a path that does not exist). Codes in the `5xx` range mean the **server** failed (a bug, an unreachable database). The client can fix a `4xx` by changing the request; only the API's owner can fix a `5xx`.

In this lab you build and deploy an API with these endpoints:

| Endpoint | Returns |
| --- | --- |
| `GET /` | `{"books": 101}`: the number of documents in `gutenberg.books` (provided) |
| `GET /books` | a list of `{"_id", "title", "author"}`, sorted by title; `?author=...` keeps one author's books |
| `GET /books/{key}` | the document of one book, without its dates; `404` if there is no such book |
| `GET /authors` | optional (Step 2.8): the names of the authors, each once, sorted |

**Where you work.** Everything runs in the **terminal on your computer**. You test the API on your computer, then deploy it to AWS; both read from your Atlas cluster.

## Step 1: Setup

<!-- TODO diagram: docs/setup.png - laptop: clone of lab-10-api with book-analyzer/ copied from the Lab 07 clone (app.py, chalicelib/analyze.py, .chalice/config.json, .chalice/deployed/dev.json); AWS (already there from Lab 07): bucket public-mst3k, Lambda book-analyzer-dev-analyze_new_book with S3 trigger, SSM mongodb_credentials; Atlas gutenberg.books (101 documents, with llm_analysis from Lab 08); MONGODB_ATLAS_* env vars on the laptop -->

### 1.1 Fork and clone this repository

Fork this repository on GitHub (keep the default name): [https://github.com/ksiller/lab-10-api](https://github.com/ksiller/lab-10-api). Clone **your fork** on your computer, in the same folder as your Lab 07 clone:

```bash
mkdir -p ~/ds2022-fall-2026/
cd ~/ds2022-fall-2026/
git clone https://github.com/YOUR_USERNAME/lab-10-api.git
cd lab-10-api
pwd
```

`pwd` should print a path ending in `ds2022-fall-2026/lab-10-api`. This README refers to that location as the *top folder* of your cloned repository.

### 1.2 Python environment

Create the environment from `pyproject.toml` and `uv.lock`, and check that Chalice is installed:

```bash
uv sync
uv run chalice --version
```

`chalice` is a development tool, as in Lab 07: it runs on your computer to test and deploy, and is not part of the deployed app.

### 1.3 AWS credentials, Atlas, and your Lab 07 resources

Your API reuses three things from Lab 07: your AWS credentials, the Lambda function `book-analyzer-dev-analyze_new_book`, and the SSM parameter `mongodb_credentials`. It also needs your Atlas cluster. Check each one now, so that a problem shows up here and not in the middle of a deploy.

Start the Learner Lab, and copy the credentials from **AWS Details** into `~/.aws/credentials` (Lab 07, Setup, Step 4).

**Your AWS credentials.** This command asks AWS who you are. It only works if the credentials in `~/.aws/credentials` are from the current lab session:

```bash
aws sts get-caller-identity
```

It prints your `UserId`, your `Account` number, and an `Arn` that contains `assumed-role`. If it prints `ExpiredToken` or `InvalidClientTokenId`, the credentials are from an earlier session: copy them again from **AWS Details**.

**Your Lab 07 Lambda function.** `chalice deploy` in Step 2.5 updates this function, so it must still exist. The command asks Lambda for the function, and `--query Configuration.FunctionName --output text` prints only its name:

```bash
aws lambda get-function --function-name book-analyzer-dev-analyze_new_book --query Configuration.FunctionName --output text
```

It prints `book-analyzer-dev-analyze_new_book`. If it prints `ResourceNotFoundException`, the function was deleted: deploy it again as in Lab 07, Part 2.

**Your SSM parameter.** Your deployed API reads the Atlas URL, user, and password from this parameter, because Lambda does not have your environment variables. The command asks SSM for the parameter, and `--query Parameter.Name` prints only its name, not the secret value:

```bash
aws ssm get-parameter --name mongodb_credentials --query Parameter.Name --output text
```

It prints `mongodb_credentials`. If it prints `ParameterNotFound`, create the parameter again (Lab 07, Setup, Step 5).

**Your Atlas cluster.** `chalice local` in Step 2.2 connects to your cluster with the three `MONGODB_ATLAS_*` variables from Lab 05, so check that this connection still works. This command connects with these variables and sends a `ping`, as in Lab 07, Setup, Step 1:

```bash
mongosh "$MONGODB_ATLAS_URL" --apiVersion 1 --username "$MONGODB_ATLAS_USER" --eval "db.runCommand({ping: 1})"
```

Enter your Atlas password when asked. The output reports `{ ok: 1 }`: your variables are set, the cluster runs, and it accepts your connection.

Troubleshooting

- **It waits, then fails with a timeout or server selection error.** Your cluster is most likely paused: Atlas pauses free clusters that have not been used for a while. Connecting with `mongosh` resumes it, so this command has already started it. Wait a few minutes, and run the command again until it reports `{ ok: 1 }`. Your data and settings stay as they are.
- **It fails with an error about the URL or the user name.** A variable is missing in this terminal. In Lab 05 you saved them in `~/.bashrc` (or `~/.zshrc`): open a new terminal, or load the file with `source ~/.bashrc` (or `source ~/.zshrc`), and try again.
- **It fails with** `Authentication failed`**.** The password does not match the database user in Atlas.

The ping comes from your computer. The API you deploy will run in Lambda, which has no fixed address, so also check in the Atlas web interface: log in at [cloud.mongodb.com](https://cloud.mongodb.com), open your project, and under **Security** → **Database & Network Access** → **IP Access List** check that the entry `0.0.0.0/0` from Lab 07 is still there and **Active**.

### 1.4 Copy your Chalice app from Lab 07

You extend the app you deployed in Lab 07, not a new one.

If your Lab 07 clone is not on this computer, clone your Lab 07 fork next to this repository first. You committed `book-analyzer/`, including `.chalice/deployed/`, in Lab 07:

```bash
git clone https://github.com/YOUR_USERNAME/lab-07-ec2-lambda.git ../lab-07-ec2-lambda
```

Copy the whole folder `book-analyzer/` from your Lab 07 clone into the top folder of this repository. **Replace the path** `../lab-07-ec2-lambda` if your Lab 07 clone is somewhere else:

```bash
cp -R ../lab-07-ec2-lambda/book-analyzer .
ls -A book-analyzer book-analyzer/.chalice
```

In the folder `book-analyzer` you should now have:

- `app.py`: your Chalice app with the Lab 07 handler `analyze_new_book()`; **you add the API routes to it in Step 2.1**
- `chalicelib/`: `analyze.py` with the analysis and the `connect()` function, which Chalice packages with your app; **no changes needed**
- `requirements.txt`: the packages Chalice installs into the Lambda function, here `pymongo`; **no changes needed**
- `.chalice/config.json`: the settings for `chalice deploy`, such as the app name and the role `LabRole`; **no changes needed**
- `.chalice/deployed/dev.json`: the record of what `chalice deploy` created in AWS in Lab 07; **no changes needed**: `chalice deploy` updates it in Step 2.5

Now check that you copied the app you deployed in Lab 07. Run each command on its own.

1. Show the bucket name in `app.py`:

   ```bash
   grep -n "BUCKET =" book-analyzer/app.py
   ```

   It prints one line such as `15:BUCKET = "public-mst3k"`, with your computing ID instead of `mst3k`. If it shows `mst3k` or another bucket, you copied the wrong folder: delete the copy with `rm -rf book-analyzer`, and copy the folder again from your Lab 07 clone, at the start of this step (Step 1.4). If that clone is not your finished Lab 07 app, clone your Lab 07 fork instead, also at the start of this step.

2. Show the line where your Lab 07 handler analyzes a new book:

   ```bash
   grep -n "analyze_book(" book-analyzer/app.py
   ```

   It prints a line such as `26:    analyze_book(event.bucket, event.key)`: the code you wrote in Lab 07, Part 2. If it prints nothing, this is not your finished Lab 07 app: delete the copy with `rm -rf book-analyzer`, and copy the folder again from your *completed* Lab 07 clone, at the start of this step (Step 1.4). If that clone is not your finished Lab 07 app, clone your Lab 07 fork instead, also at the start of this step.

3. Compare the copied `analyze.py` with the one in this repository:

   ```bash
   diff book-analyzer/chalicelib/analyze.py src/book_analysis/analyze.py && echo 'Good, the `analyze.py` is identical in both places and ready for your edits'
   ```

   It should print ``Good, the `analyze.py` is identical in both places and ready for your edits``. If it prints lines with `<` and `>`, the two files differ: copy `src/book_analysis/analyze.py` to `book-analyzer/chalicelib/analyze.py`.

## Step 2: Build the API with Chalice

<!-- TODO diagram: docs/api.png - client (curl, browser) -> HTTPS GET /api/books/{key} -> API Gateway API book-analyzer (stage api) -> Lambda book-analyzer-dev (handler app.app, routes index, list_books, get_book; role LabRole) -> SSM mongodb_credentials -> Atlas gutenberg.books (find, projection) -> JSON back to the client; next to it, unchanged from Lab 07: S3 public-mst3k -> book-analyzer-dev-analyze_new_book -> Atlas; laptop: chalice local on 127.0.0.1:8000 with MONGODB_ATLAS_* for testing -->

**What we need:** The API needs two new parts in AWS.

- an **API in Amazon API Gateway**, which gives the API its public URL, receives each request, and passes it to a Lambda function
- a **second Lambda function**, which answers the requests: it reads from MongoDB and returns JSON

You do not create them by hand. You add the API functionality to your Lab 07 Chalice app, and `chalice deploy` creates both the needed API Gateway and the new Lambda function. In Lab 07 you wrote one handler, `analyze_new_book()`, with the decorator `@app.on_s3_event`: S3 events run it. Now you add handlers with the decorator `@app.route`: HTTP requests run them. Chalice puts all route handlers into **one** new Lambda function, `book-analyzer-dev`, and creates the API in API Gateway in front of it. Your Lab 07 function stays as it is.

Work in the `book-analyzer/` folder for Steps 2.1 to 2.6:

```bash
cd book-analyzer
```

### 2.1 Add the routes to `app.py`

Open `book-analyzer/app.py`. Keep everything that is there: your Lab 07 handler must still work after the deploy. Make two changes.

**1. Change the two imports** at the top, so that they read:

```python
from chalice import Chalice, NotFoundError

from chalicelib.analyze import COLLECTION, DATABASE, analyze_book, connect
```

`NotFoundError` is the error that Chalice turns into a `404` response. The `connect()` function and the names `DATABASE` (`"gutenberg"`) and `COLLECTION` (`"books"`) come from the same `analyze.py` that your Lambda function already uses: `connect()` reads the Atlas settings from your `MONGODB_ATLAS_*` variables on your computer, and from the SSM parameter `mongodb_credentials` in Lambda.

**2. Add this code at the end of the file**, below the `analyze_new_book()` function:

```python
# Lab 10: the API routes. They only read from MongoDB; none of them changes your data.

# Projection for get_book() (Step 2.4): all fields except runs and llm_analysis.at.
# They hold dates, and Chalice cannot turn a date into JSON.
BOOK_FIELDS = {"runs": 0, "llm_analysis.at": 0}


@app.route("/")
def index():
    """Return the number of books in the collection."""
    client = connect()
    try:
        count = client[DATABASE][COLLECTION].count_documents({})
    finally:
        client.close()  # close the connection to Atlas, also after an error
    return {"books": count}


@app.route("/books")
def list_books():
    """Return _id, title, and author of the books, sorted by title.

    The query parameter author, for example /books?author=Mark%20Twain, keeps only that author's books.
    """
    # TODO Step 2.3: replace this line with your code (see the README)


@app.route("/books/{key}")
def get_book(key):
    """Return the document of one book, for example /books/pg11.txt; 404 if there is none."""
    # TODO Step 2.4: replace this line with your code (see the README)
```

The `index()` function is complete; it is the model for the other two. What the provided parts do:

| Part | What it does |
| --- | --- |
| `@app.route("/")` | connects the function to the path `/`. Without `methods=[...]`, a route answers `GET` requests only |
| `client = connect()` ... `finally: client.close()` | opens a connection to Atlas for this request and closes it afterwards, also after an error, as in Lab 07 |
| `count_documents({})` | counts the documents that match the empty filter `{}`, so all of them (Lab 05) |
| `return {"books": count}` | Chalice turns the returned dictionary or list into a JSON response with status `200` |
| `BOOK_FIELDS` | a projection you use in the `get_book()` function in Step 2.4; the `index()` and `list_books()` functions don't need it |

### 2.2 Test on your computer with `chalice local`

`chalice local` runs your routes in a small web server on your computer, without deploying anything. It uses your `MONGODB_ATLAS_*` variables to reach Atlas. In `book-analyzer/`:

```bash
uv run chalice local
```

It prints `Serving on http://127.0.0.1:8000` and keeps running. **Open a second terminal**, change into your cloned repository, and call the endpoint with `curl`:

```bash
cd ~/ds2022-fall-2026/lab-10-api
curl http://127.0.0.1:8000/
```

```json
{"books":101}
```

You can also open [http://127.0.0.1:8000/](http://127.0.0.1:8000/) in your browser. The first terminal logs one line per request, with its status code. `chalice local` reloads your code when you save `app.py`, so leave it running while you work on Steps 2.3 and 2.4. Stop it with **Ctrl+C** when you are done.

> [!NOTE]
> The number is 101 if you have the 100 books from Lab 06 and `pg74.txt` from Lab 07. If you uploaded more books, it is higher.

### 2.3 Write the `list_books()` function

Replace the `TODO` line in the `list_books()` function with code that returns a list of books, one dictionary per book, with only `_id`, `title`, and `author`, sorted by title. If the URL has the query parameter `author`, it returns only that author's books.

| Requirement | Hint |
| --- | --- |
| connect, and close in `finally` | as in the `index()` function |
| all books, three fields | `find(filter, projection)`; the projection `{"title": 1, "author": 1}` keeps these two fields and `_id` |
| sorted by title | `.sort("title", 1)`, as `sort` in Lab 07, Part 3 |
| a list, not a cursor | `find()` returns a cursor, an object that fetches the documents one at a time, which Chalice cannot turn into JSON; `list(cursor)` reads all documents into a list |
| `?author=...` | `app.current_request.query_params` is a dictionary of the query parameters, or `None` if the URL has none. If it has the key `"author"`, add `"author": value` to the filter |

Test it while `chalice local` runs, in your second terminal:

```bash
curl http://127.0.0.1:8000/books
```

```bash
curl "http://127.0.0.1:8000/books?author=Mark%20Twain"
```

The first prints all your books, starting with the titles that begin with `A`; the second prints only the books by Mark Twain, among them `pg74.txt` from Lab 07:

```json
[{"_id":"pg74.txt","author":"Mark Twain","title":"The Adventures of Tom Sawyer, Complete"},{"_id":"pg1837.txt","author":"Mark Twain","title":"The Prince and the Pauper"}]
```

Put a URL with `?` or `&` in quotes: unquoted, your shell would treat these characters itself. `%20` is a space; a URL cannot contain a space.

**Optional: pretty-print with `jq`.** `curl -s http://127.0.0.1:8000/books | jq '.[].title'` prints only the titles; `jq length` counts the books.

### 2.4 Write the `get_book()` function

Replace the `TODO` line in the `get_book()` function with code that returns the document of the book whose `_id` is `key`, without the date fields, or a `404` error if there is no such document.

**Document structure in your Mongo database:** Each document in `gutenberg.books` holds the analysis of one book: `_id` (the S3 key, such as `pg11.txt`), `bucket`, `title`, `author`, `language`, `word_count`, `unique_words`, and `vocabulary_ratio` from Lab 07; `runs`, a list with one entry for each time the analysis ran (Labs 07 and 08), each with its date and time in `at`; and `llm_analysis` from Lab 08, with `summary`, `genre`, `model`, and its date and time in `at`. JSON has strings, numbers, `true`/`false`, `null`, lists, and objects, but no type for a date. Chalice cannot turn the dates in `runs` and `llm_analysis.at` into JSON, and fails with a `500`. So your `get_book()` function must leave these two fields out.

**Projection to remove date fields:** A projection is the second argument of `find()` and `find_one()`, which selects the fields a query returns ([Class 05: Project only some fields](https://github.com/ksiller/DS2022/blob/main/class/05-nosql/README.md#project-only-some-fields)). `1` keeps a field, and `0` leaves it out, as `_id: 0` does in the Class 05 example. The projection you need is already defined in the variable `BOOK_FIELDS` (Step 2.1): `{"runs": 0, "llm_analysis.at": 0}` returns every field except these two. The `list_books()` function does not need it, because its projection keeps only `title`, `author`, and `_id`.

| Requirement | Hint |
| --- | --- |
| one document by its `_id` | `find_one(filter, projection)` returns the document, or `None` if none matches |
| without the dates | use `BOOK_FIELDS` as the projection |
| `404` if there is no such book | `raise NotFoundError(f"No book with key {key}")`; Chalice turns it into a response with status `404` |

The path parameter arrives as the function's argument: for `/books/pg11.txt`, Chalice calls `get_book("pg11.txt")`, because the route says `{key}` and the function's parameter is named `key`.

Test it. `-i` makes `curl` print the status line and the headers of the response before the JSON:

```bash
curl -i http://127.0.0.1:8000/books/pg11.txt
```

```text
HTTP/1.1 200 OK
Content-Type: application/json
...
{"_id":"pg11.txt","author":"Lewis Carroll","bucket":"public-mst3k","language":"English","title":"Alice's Adventures in Wonderland","unique_words":2575,"vocabulary_ratio":0.0939,"word_count":27427,"llm_analysis":{"summary":"Alice falls down a rabbit hole, ...","genre":"adventure","model":"Qwen/Qwen2.5-1.5B-Instruct"}}
```

And for a book that does not exist:

```bash
curl -i http://127.0.0.1:8000/books/pg99999.txt
```

```text
HTTP/1.1 404 Not Found
...
{"Code":"NotFoundError","Message":"No book with key pg99999.txt"}
```

If a book has no `llm_analysis`, Lab 08 did not finish it; the API still returns the rest of its document.

> [!TIP]
> If you get stuck, ask Claude Code to check your handler, for example: *"I wrote `get_book` in `book-analyzer/app.py`. Check it against the requirements in Step 2.4 of the README and tell me what is wrong, without fixing it."* (see [Using Claude Code](#using-claude-code)).

When both endpoints work, stop `chalice local` with **Ctrl+C** in the first terminal.

### 2.5 Deploy

In `book-analyzer/`:

```bash
uv run chalice deploy
```

`chalice deploy` reads `.chalice/deployed/dev.json`, copied in Step 1.4, to find what it created in Lab 07. That is why it **updates** your Lab 07 function instead of creating it again. Without `dev.json`, it would try to create the function again and fail, because the function already exists (see [Troubleshooting](#troubleshooting)).

Chalice packages `app.py`, `chalicelib/`, and `pymongo` as in Lab 07, updates your Lab 07 function, and creates the new parts. The end of its output lists them:

```text
Resources deployed:
  - Lambda ARN: arn:aws:lambda:us-east-1:123456789012:function:book-analyzer-dev-analyze_new_book
  - Lambda ARN: arn:aws:lambda:us-east-1:123456789012:function:book-analyzer-dev
  - Rest API URL: https://abc123defg.execute-api.us-east-1.amazonaws.com/api/
```

| Resource | What it is |
| --- | --- |
| `book-analyzer-dev-analyze_new_book` | your Lab 07 function, updated in place; S3 still triggers it |
| `book-analyzer-dev` | **new**: one Lambda function for all three routes. Its name has no function name at the end, as your Lab 07 function has, because it runs every `@app.route` handler |
| `Rest API URL` | **new**: the public URL of your API in API Gateway (Chalice calls this kind of API a *REST API*). `/api/` at the end is the stage, as in [APIs, endpoints, and requests](#apis-endpoints-and-requests) |

`uv run chalice url` prints the URL again at any time. Save it in a variable `API` for the next commands, and in a file for your submission:

```bash
mkdir -p ../results
uv run chalice url > ../results/api-url.txt
API=$(cat ../results/api-url.txt)
echo $API
```

`echo` prints your URL, ending in `/api/`. The variable `API` exists only in this terminal; in a new terminal, run the last two lines again.

You upload `results/api-url.txt` to Canvas in Step 3.3. Git does not commit it: `.gitignore` leaves it out, because your fork on GitHub is public, and the URL should not be.

In the **AWS Console**, check:

- **The API:** **API Gateway** → **APIs** → `book-analyzer`. Under **Resources** you see your three paths, `/`, `/books`, and `/books/{key}`, each with a `GET` method.
- **The function:** **Lambda** → **Functions** → `book-analyzer-dev`. Its **Function overview** shows **API Gateway** as the trigger.

### 2.6 Call your API in the cloud

Call the same endpoints as in Steps 2.2 to 2.4, now at your API's URL. `$API` ends in `/`, so the paths follow without one:

```bash
curl ${API}
```

```bash
curl "${API}books?author=Mark%20Twain"
```

```bash
curl -i ${API}books/pg74.txt
```

The answers are the same as from `chalice local`, but now they come from Lambda: the function read your Atlas settings from SSM, because Lambda does not have your environment variables. The first call after a deploy, or after a few idle minutes, takes a second or two longer: Lambda first has to start your function (a **cold start**).

Now call a path that your API does not have, here `book` instead of `books`:

```bash
curl -i ${API}book/pg74.txt
```

```text
HTTP/2 403
...
{"message":"Missing Authentication Token"}
```

API Gateway answers this itself, without running your function: it has no route `/book/{key}`. The message is misleading; it does not mean that your API needs a login. It is how API Gateway reports a path or method it does not know.

Open `${API}books/pg74.txt` in your browser too: any browser can call your API, on any computer. Then save two responses for your submission:

```bash
curl -s ${API}books > ../results/book-list.json
curl -s ${API}books/pg74.txt > ../results/single-book.json
jq length ../results/book-list.json
```

`jq length` prints `101`, the number of books in the list.

**Read the log.** Every request is one run of `book-analyzer-dev`, with a log in CloudWatch, as in Lab 07. For the function behind the routes, Chalice uses the name `api_handler`:

```bash
uv run chalice logs -n api_handler
```

If a request returns `{"message": "Internal server error"}` or `{"Code":"InternalServerError",...}`, the log shows the Python error behind it (see [Troubleshooting](#troubleshooting)).

**Success:** Your book analysis is available at a public URL, as JSON, served by a Lambda function that runs only when someone calls it.

Go back to the top folder of your cloned repository:

```bash
cd ..
```

> [!WARNING]
> Your API is **open**: anyone who has the URL can call it, without a password. It only reads, and it returns only data about books whose copyright has expired, so nobody can change or delete anything through it. Every call still runs your Lambda function and opens a connection to your Atlas cluster. Do not post the URL publicly. Leave the API deployed until your lab is graded; [Submit your work](#step-3-submit-your-work) says how to remove it afterwards.

### 2.7 Add a new book and find it in your API

Your API reads the database on every request, so a new book appears in it as soon as your Lab 07 function has analyzed it. In this step you add one more book and follow it from S3 to your API. Work in the top folder of your cloned repository, and run each command on its own.

1. **Pick a book that is not in your collection.** Find one on [gutenberg.org](https://www.gutenberg.org): its number is at the end of its web address, for example `1342` in `https://www.gutenberg.org/ebooks/1342`. The commands below use `1342`; replace it with your book's number in every command. Check that your API does not have the book yet:

   ```bash
   curl -i ${API}books/pg1342.txt
   ```

   It should print `HTTP/2 404` and `No book with key pg1342.txt`. If it prints `200`, the book is already in your collection: pick another one.

2. **Download the book**, as in Lab 07, Part 2, Step 7. Download single books by hand only: Project Gutenberg does not allow automated bulk downloads.

   ```bash
   curl -sSo pg1342.txt https://www.gutenberg.org/cache/epub/1342/pg1342.txt
   ```

3. **Upload it to your bucket.** Replace `mst3k` with your computing ID:

   ```bash
   aws s3 cp pg1342.txt s3://public-mst3k/
   ```

   The upload triggers your Lab 07 function `analyze_new_book`, which analyzes the book and saves its document in Atlas.

4. **Wait a few seconds, then count the books:**

   ```bash
   curl ${API}
   ```

   It should print one more book than before, for example `{"books":102}`. If the number has not changed, wait a few seconds and try again. If it still does not change, read the function's log in `book-analyzer/` with `uv run chalice logs -n analyze_new_book` (see [Troubleshooting](#troubleshooting)).

5. **Find the new book in the list.** `jq` keeps only the entry whose `_id` is your book:

   ```bash
   curl -s ${API}books | jq '.[] | select(._id == "pg1342.txt")'
   ```

   It should print the `_id`, `title`, and `author` of your book. The `_id` is the name of the file you uploaded: your Lab 07 function uses the S3 key as the document's `_id`.

6. **Get the book's document with this `_id`:**

   ```bash
   curl -i ${API}books/pg1342.txt
   ```

   It should print `HTTP/2 200` and the document, with `word_count`, `unique_words`, and `vocabulary_ratio`, but without `llm_analysis`: the summary and the genre come from Lab 08, which did not run on this book. Save the document for your submission:

   ```bash
   curl -s ${API}books/pg1342.txt > results/new-book.json
   ```

### 2.8 (optional) Add a `GET /authors` endpoint

The `?author=` filter of `/books` only matches a name exactly as it is stored, for example `Mark Twain`. An endpoint that lists every author once shows which names your collection has. Work in `book-analyzer/` (`cd book-analyzer`).

Add this route at the end of `book-analyzer/app.py`:

```python
@app.route("/authors")
def list_authors():
    """Return the names of the authors in the collection, each once, sorted."""
    # TODO Step 2.8 (optional): replace this line with your code (see the README)
```

Replace the `TODO` line in the `list_authors()` function with code that returns a list of the author names, each name once, sorted.

| Requirement | Hint |
| --- | --- |
| connect, and close in `finally` | as in the `index()` function |
| each author once | the method `distinct("author")` returns a list of the different values of a field, as `books.distinct("runs.host")` in `report.py` of Lab 07 |
| sorted | MongoDB does not promise an order for `distinct()`; the Python function `sorted()` sorts the list |

Test it with `uv run chalice local` and `curl http://127.0.0.1:8000/authors` (Step 2.2), then deploy again (`uv run chalice deploy`, Step 2.5) and call your API:

```bash
curl -s ${API}authors | jq length
```

It prints the number of different authors. Pick a name from `curl -s ${API}authors` and use it with the `?author=` filter (Step 2.3), with `%20` for each space. Save the list for your submission:

```bash
curl -s ${API}authors > ../results/authors.json
cd ..
```

### Milestone

- [ ] `results/api-url.txt` contains your API's URL, ending in `/api/`.
- [ ] `curl -i ${API}books/pg74.txt` returns `200` and the document of *The Adventures of Tom Sawyer*; `curl -i ${API}books/pg99999.txt` returns `404`.
- [ ] `results/book-list.json` lists all your books, and `results/single-book.json` holds the document of `pg74.txt`.
- [ ] The book you added in Step 2.7 appears in `${API}books`, and `results/new-book.json` holds its document.
- [ ] Optional: `results/authors.json` lists your authors (Step 2.8).

**Congratulations, you did it! As a last step, complete the [Knowledge check](#knowledge-check) below and [submit](#step-3-submit-your-work).**


## Knowledge check

The questions are in [KNOWLEDGE_CHECK.md](KNOWLEDGE_CHECK.md), in the top folder of your cloned repository. Open it, and write your answer below each question, in your own words:

1. **API Gateway:** follow a request from your API's URL to the Lambda function that answers it (screenshot: `results/api-gateway.png`)
2. **Status codes:** who has to fix a `403`, a `404`, and a `500`
3. **An open URL:** what an open, read-only API risks, and one way to limit it
4. **Chalice or FastAPI?** your Chalice API or a FastAPI server as in Class 10, for a given case, and why

## Step 3: Submit your work

### 3.1 Check your files

Submit from the top folder of your clone. Do not change `README.md`. Your repository should look like this:

```text
lab-10-api/
├── book-analyzer/          # your Lab 07 app, with the three API routes you added
├── KNOWLEDGE_CHECK.md      # your answers
├── results/
│   ├── api-url.txt     # not committed: you upload it to Canvas
│   ├── book-list.json
│   ├── single-book.json
│   ├── new-book.json
│   ├── authors.json        # optional
│   └── api-gateway.png
└── ... (the provided files, unchanged)
```

| File | From |
| --- | --- |
| `results/api-url.txt` | Step 2.5 (upload to Canvas, not committed) |
| `results/book-list.json`, `results/single-book.json` | Step 2.6 |
| `results/new-book.json` | Step 2.7 |
| `results/authors.json` (optional) | Step 2.8 |
| `results/api-gateway.png` | Knowledge check, question 1 |

### 3.2 Commit and push

Stage your files, and check what is staged:

```bash
git add book-analyzer/ KNOWLEDGE_CHECK.md results/
```

```bash
git status
```

`git status` must not list a `.pem` file, an `.aws/` folder, a `.env` file, or a `.bak` file as staged. If it does, take the file out with `git restore --staged <file>`. `git status` must not list `results/api-url.txt` or `book-analyzer/.chalice/deployed/` either: both contain your API's URL, and `.gitignore` leaves them out. `book-analyzer/.chalice/config.json` contains your account ID; that is not a secret and can be pushed to GitHub. Then commit and push:

```bash
git commit -m "Complete Lab 10: API"
git push origin main
```

### 3.3 Submit on Canvas

In the Canvas assignment:

1. Submit the URL of your fork: `https://github.com/YOUR_USERNAME/lab-10-api`.
2. Upload the file `results/api-url.txt` with your API's URL.

> **Keep your API deployed until your lab is graded:** the graders call the URL in the `api-url.txt` you uploaded to Canvas. After your grade is posted, remove the API and both functions, in `book-analyzer/`, with `uv run chalice delete`. This also removes your Lab 07 function and its S3 trigger; your bucket, your books, and your documents in Atlas stay. Do not click **Reset** in the Learner Lab.

## Using Claude Code

You may use [Claude Code](https://docs.claude.com/en/docs/claude-code/overview) for this lab. Open a new terminal, change into your cloned repository, and start it:

```bash
cd ~/ds2022-fall-2026/lab-10-api
claude
```

Claude Code reads [CLAUDE.md](CLAUDE.md) in this repository: instructions that tell it about the lab, the Learner Lab, and your Atlas cluster, so you do not have to explain them yourself.

**In this repository, Claude Code works as your teaching assistant (TA).** `CLAUDE.md` tells it to explain, to help you find the cause of an error, and to give you hints, but not to write the graded parts for you: the bodies of the `list_books()` and `get_book()` functions in `book-analyzer/app.py`, and your answers in `KNOWLEDGE_CHECK.md`. If you ask it for one of these, it says so and gives you the next hint instead. That is intended, not a malfunction.

The goal is that you can build, test, and debug an API, not that Claude Code can. Ask it to explain before it changes anything, and read every `aws`, `chalice`, and `curl` command before you let it run one.

**Prompts for building the API:**

- "Explain what `@app.route` does in Chalice, and how it differs from `@app.on_s3_event` in my Lab 07 handler. Don't change anything."
- "How do I read a query parameter in a Chalice route? Show me with a route that is not part of my lab."
- "Why does `find()` need `list()` around it before Chalice can return it?"
- "I wrote `get_book` in `book-analyzer/app.py`. Check it against the requirements in Step 2.4 of the README and tell me what is wrong, without fixing it."
- "What does `chalice deploy` create in AWS for my routes, and why is there a second Lambda function?"

**Prompts for troubleshooting:**

- "My deployed API returns `Internal server error`. Here is the output of `uv run chalice logs -n api_handler`: [paste]. What is the cause? Explain it before you suggest a fix."
- "`chalice deploy` failed with: [paste]. Is this a problem in my code, in `.chalice/`, or in the Learner Lab's permissions?"
- "`curl` with `?author=Mark Twain` returns all books or an error. Why?"

**Never paste** your Atlas password, the contents of `~/.aws/credentials`, or the output of `printenv` into a prompt. Do not let Claude Code answer the questions in `KNOWLEDGE_CHECK.md` for you: they are graded on your own understanding.

## Troubleshooting

**Setup and `chalice deploy`**

- `chalice deploy`**:** `ResourceConflictException` **or** `Function already exist`**.** Chalice does not know about your Lab 07 deployment: `book-analyzer/.chalice/deployed/dev.json` is missing, because the folder was not copied from your Lab 07 clone (Step 1.4). Copy the folder again from your Lab 07 clone or your Lab 07 fork. If you no longer have `dev.json` anywhere, delete the function as in Lab 07, Troubleshooting (**Lambda** → `book-analyzer-dev-analyze_new_book` → **Delete**, and the bucket's event notification), then deploy again.
- `chalice deploy`**:** `iam:CreateRole` **or** `ACCOUNT_ID` **in an error message.** `book-analyzer/.chalice/config.json` is not the one from Lab 07. Copy the folder again (Step 1.4).
- `ExpiredToken` **or** `InvalidClientTokenId`**.** Your lab session changed. Paste a fresh block from **AWS Details** → **AWS CLI** into `~/.aws/credentials`.
- `Address already in use` **from** `chalice local`**.** Another server still runs on port 8000, for example the FastAPI server from Class 10. Stop it with **Ctrl+C** in its terminal, or start this one on another port with `--port 8001` (and use that port in your URLs).

**Your routes**

- `TypeError: Object of type datetime is not JSON serializable`**.** The response contains a date: `runs` or `llm_analysis.at`. Use `BOOK_FIELDS` as the projection in the `get_book()` function (Step 2.4); the `list_books()` function should keep only `title` and `author`.
- `TypeError: Object of type Cursor is not JSON serializable`**.** The handler returns the cursor from `find()`. Return `list(...)` of it.
- `TypeError: argument of type 'NoneType' is not iterable` **in the** `list_books()` **function.** The URL has no query parameters, so `app.current_request.query_params` is `None`. Use `app.current_request.query_params or {}`.
- `null` **instead of** `404` **for a missing book.** `find_one()` returned `None`, and the handler returned it. Check for `None` and raise `NotFoundError`.
- `{"message":"Missing Authentication Token"}` **(status** `403`**).** API Gateway has no route for this path or method: a typo in the path, `/api` missing after the server name, or a double `//` (`$API` already ends in `/`). Check the URL against `uv run chalice url` and your routes.
- **The author filter returns all books, or none.** The query parameter is not in quotes or not encoded: use `"...books?author=Mark%20Twain"`, in quotes, with `%20` for the space. The value must match the `author` field exactly, also in case.
- `{"Code":"InternalServerError","Message":"An internal server error occurred."}` **or** `{"message": "Internal server error"}` **from the deployed API.** Your function raised an error. `uv run chalice logs -n api_handler` (in `book-analyzer/`) shows it. If the same request works with `chalice local`, the difference is where it runs (Lambda instead of your computer): see **MongoDB Atlas** below.

**MongoDB Atlas**

- `ServerSelectionTimeoutError`**, or** `Task timed out after 60.00 seconds` **in the log.** Your Atlas cluster is paused, or `0.0.0.0/0` is missing from the IP Access List. To resume the cluster, connect with `mongosh` as in [Step 1.3](#13-aws-credentials-atlas-and-your-lab-07-resources), wait a few minutes, and try again. The deployed API runs in Lambda, which has no fixed address.
- `Cannot read SSM parameter mongodb_credentials` **in** `chalice logs`**.** The parameter is missing: create it again (Lab 07, Setup, Step 5). `chalice local` does not need it, because it uses your environment variables.
- `Authentication failed`**.** The user or password in `mongodb_credentials` (in Lambda) or in your environment variables (on your computer) does not match the database user in Atlas.
- `KeyError: 'MONGODB_ATLAS_USER'`**.** Only some of the three variables are set in this terminal. Open a new terminal, or load `~/.bashrc` (or `~/.zshrc`) with `source`, and run the connection check from [Step 1.3](#13-aws-credentials-atlas-and-your-lab-07-resources) again.

## Resources

**This course**

- [Class 10: FastAPI](https://github.com/ksiller/DS2022/blob/main/class/10-fastapi/README.md): a Book API with FastAPI, `curl`, and the generated documentation
- [Lab 07, Part 2](https://github.com/ksiller/lab-07-ec2-lambda#part-2-build-a-lambda-app-for-new-books): the Chalice app you extend

**Chalice and API Gateway**

- [Chalice quickstart: deploying an API](https://aws.github.io/chalice/quickstart.html#deploying)
- [Chalice: Routing](https://aws.github.io/chalice/topics/routing.html): path parameters, query parameters, `current_request`, error classes
- [Chalice: Testing](https://aws.github.io/chalice/topics/testing.html) (the test client) and [configuration file](https://aws.github.io/chalice/topics/configfile.html)
- [Amazon API Gateway: developer guide](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-rest-api.html) and [limits on requests](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-throttling.html)

**HTTP**

- [MDN: An overview of HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Overview) and [HTTP response status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status)
- [curl man page](https://curl.se/docs/manpage.html): `-i` and `-s`
- [PyMongo tutorial](https://pymongo.readthedocs.io/en/stable/tutorial.html): `find`, `find_one`, projections, and `sort`

## Limitations of this approach

- Every request opens a new connection to Atlas, and in Lambda also reads the SSM parameter, then closes both. Many requests at once would mean many connections to a free cluster. Connecting once and reusing the connection for later requests would be faster.
- The API is open: no API key, no user accounts, no limit on requests. Real APIs often require an API key (in Chalice: `api_key_required=True`) and set rate limits, as the APIs from class do.
- `GET /books` returns all books in one response. With thousands of books, an API returns them in pages (`?skip=` and `?limit=`).
- The API leaves out the dates. A real API converts them to text, such as `"2026-11-12T15:20:41Z"`, which FastAPI (Class 10) does automatically and Chalice does not.
- There are no automated tests. Chalice has a test client (`chalice.test.Client`) that calls your routes without a server, so that they can be checked automatically after every change.
