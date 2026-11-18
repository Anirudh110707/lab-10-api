# Lab 10: Knowledge check

**Answer each question below it, in your own words, in one to three sentences.** Replace the `...` below **Your answer:** with your text.

## 1. From the URL to the function

A classmate calls `${API}books/pg74.txt` and gets an error. To help, you need to know which Lambda function answered the request. In the **AWS Console**, open **API Gateway** → **APIs** → `book-analyzer` → **Resources**, click the `GET` method under `/books/{key}`, and look at its **Integration request**: the setting that tells API Gateway where to pass the request. Take a screenshot that shows it and the Lambda function it names, and save it as `results/api-gateway.png`.

**Your answer:** Which Lambda function answers the request? Why did Chalice create this function instead of using `book-analyzer-dev-analyze_new_book` from Lab 07?

...

## 2. Who has to fix it?

You saw two error status codes in this lab: `403` with `Missing Authentication Token` for `${API}book/pg74.txt`, and `404` for `${API}books/pg99999.txt`. A third, `500` (or `Internal server error`), comes back when a handler fails, for example when the Atlas cluster is paused; you may have seen it too.

**Your answer:** For each of the three, say whether the client or the API's owner has to fix it, and what they would change.

...

## 3. An open URL

Your Chalice API has no password or key: anyone who has the URL can call it.

**Your answer:** It only reads data about books whose copyright has expired, so what could still go wrong if the URL were posted on a public website? Name one way to limit that risk.

...

## 4. Chalice or FastAPI?

In Class 10 you ran a Book API with FastAPI on your computer; in this lab your API runs in AWS, with Chalice. A research group wants an API for a notebook that they run a few times a week. Another team wants an API that answers thousands of requests a minute, all day.

**Your answer:** For each team, would you start from a FastAPI server like the one in Class 10 or from your Chalice API, and why? Think of where each one runs, what it costs while nobody calls it, and what happens when your computer is off.

...
