---
title: "What I Learnt About Pooling"
datePublished: Mon Dec 08 2025 15:54:06 GMT+0000 (Coordinated Universal Time)
cuid: cmixc0bca000302l6aacmchwb
slug: what-i-learnt-about-pooling
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/MVLhRT1H8Bs/upload/dabfe8ae768ab6bb8c8bfb02e305bf8a.jpeg
tags: postgresql, databases, apis, testing, curiosity, Go

---

I am working on an API and I got bored so I decided to pause and test how much load it can handle. I used ab (ApacheBench), a CLI tool to perform the tests. My goal was to see how database querying will affect performance of the API.

## Disclaimer

The endpoint used for this test is lean and may not be reflective of production performance. However, it provides an idea of the impact of different ways to solving the same problem. The magnitude of the differences are significant during testing.

## Test #1: Open and Close Connection

This was a method I learned from school, tutorials and online guides during my years of programming. You open a connection to a database, you make your query, then you close the connection. Standard and straightforward.

Endpoint used for testing:

```go
// handlers/health.go
func HealthCheck(w http.ResponseWriter, r *http.Request) *types.ErrorDetails {
	// check if loading configurations are successful
	_, err := config.LoadDBConfig()
	if err != nil {
		types.ReturnJSON(w, types.ServiceUnavailableErrorResponse())
		return &types.ErrorDetails{
			Trace:   err,
			Message: "Failed to load database config",
		}
	}

	// check if database connection is established
	ctx := r.Context()
	db, err := database.NewConnection(ctx)
	if err != nil {
		types.ReturnJSON(w, types.ServiceUnavailableErrorResponse())
		return &types.ErrorDetails{
			Trace:   err,
			Message: "Failed to establish database connection",
		}
	}

	if err := db.Ping(ctx); err != nil {
		types.ReturnJSON(w, types.ServiceUnavailableErrorResponse())
		return &types.ErrorDetails{
			Trace:   err,
			Message: "Failed to ping database",
		}
	}

	// check if jwt config is loaded successfully
	_, err = config.LoadJWTConfig()
	if err != nil {
		types.ReturnJSON(w, types.ServiceUnavailableErrorResponse())
		return &types.ErrorDetails{
			Trace:   err,
			Message: "Failed to load jwt config",
		}
	}

	return types.ReturnJSON(w, types.OKResponse("Service is up", nil))
}
```

How DB was connected to:

```go
// database/db.go
func NewConnection(ctx context.Context) (*pgx.Conn, error) {
	config, err := config.LoadDBConfig()
	if err != nil {
		return nil, err
	}

	dburl := fmt.Sprintf(
		"postgres://%s:%s@%s:%d/%s?search_path=%s",
		config.User, config.Password, config.Host, config.Port, config.DBName, config.Schema,
	)
	return pgx.Connect(ctx, dburl)
}
```

Testing the throughput with **10,000 requests** hitting the endpoint at the same time:

```bash
# command for testing

ab -n 10000 http://localhost:8000/health > results.txt
```

```json
// API logs on failed responses
{
  "time":"2025-12-08T13:46:51.918468793Z",
  "level":"ERROR",
  "msg":"Failed to establish database connection",
  "trace":"failed to connect to `user=admin database=local`:\n\t127.0.0.1:5432 (127.0.0.1): tls error: server refused TLS connection\n\t127.0.0.1:5432 (127.0.0.1): server error: FATAL: sorry, too many clients already (SQLSTATE 53300)",
  "method":"GET",
  "url":"/health",
  "remote_addr":"127.0.0.1:59034",
  "duration_ms":15
}
```

From the log, we can see that the API failed to establish a database connection because there are too many clients already.

```plaintext
# results.txt

This is ApacheBench, Version 2.3 <$Revision: 1923142 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient)


Server Software:        
Server Hostname:        localhost
Server Port:            8000

Document Path:          /health
Document Length:        40 bytes

Concurrency Level:      1
Time taken for tests:   225.952 seconds
Complete requests:      10000
Failed requests:        6300
   (Connect: 0, Receive: 0, Length: 6300, Exceptions: 0)
Non-2xx responses:      6300
Total transferred:      1664100 bytes
HTML transferred:       337000 bytes
Requests per second:    44.26 [#/sec] (mean)
Time per request:       22.595 [ms] (mean)
Time per request:       22.595 [ms] (mean, across all concurrent requests)
Transfer rate:          7.19 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        1    4   2.7      3      87
Processing:     8   19   9.9     16     252
Waiting:        6   18   9.7     15     251
Total:         10   22  11.5     20     330

Percentage of the requests served within a certain time (ms)
  50%     20
  66%     23
  75%     25
  80%     27
  90%     30
  95%     36
  98%     48
  99%     60
 100%    330 (longest request)
```

For the test results, out of 10,000 requests sent at a time, **6,300 of them failed (63%).** That is terrible. For out throughput, we have an average of **44 requests processed per second** at an average of **22.6 ms spent on each request.** It took **225.95 seconds** to process all 10,000 requests.

So, I realized I have written terrible code. If this is a ping of the database, you can imagine if it was an actual query being run. It will be much worse.

Will the results change if I run the requests concurrently instead of a single process? I will simulate different users making requests at the same time as opposed to one user making 10,000 requests. Here are the results for 100 concurrent users without changing the code:

```bash
# command for testing

ab -n 10000 -c 100 http://localhost:8000/health > results.txt
```

```plaintext
# results.txt

This is ApacheBench, Version 2.3 <$Revision: 1923142 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient)


Server Software:        
Server Hostname:        localhost
Server Port:            8000

Document Path:          /health
Document Length:        40 bytes

Concurrency Level:      100
Time taken for tests:   85.868 seconds
Complete requests:      10000
Failed requests:        7100
   (Connect: 0, Receive: 0, Length: 7100, Exceptions: 0)
Non-2xx responses:      7100
Total transferred:      1669700 bytes
HTML transferred:       329000 bytes
Requests per second:    116.46 [#/sec] (mean)
Time per request:       858.677 [ms] (mean)
Time per request:       8.587 [ms] (mean, across all concurrent requests)
Transfer rate:          18.99 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        1    7  13.8      4     202
Processing:    94  848 253.8    790    2503
Waiting:       16  847 253.4    788    2502
Total:         95  855 253.1    795    2513

Percentage of the requests served within a certain time (ms)
  50%    795
  66%    897
  75%    959
  80%   1003
  90%   1181
  95%   1301
  98%   1429
  99%   1788
 100%   2513 (longest request)
```

If we had **100 different users** making multiple requests totalling 10,000, we don't see much difference. In fact, the **failed responses increased to** **7,100 (71%**). Instead of 44, we got **116 requests processed per second**. We see Go's concurrency being leveraged. The time spent on each request because of concurrency is **8.59 ms**. It took 85.87 seconds to complete processing 10,000 requests.

With multiple users we see more requests processed per time with lower latency. But the drawback of increased failed requests is not much of a gain. We just failed faster.

## Test #2: Using Pooling For DB Connection

I went to ChatGPT to discuss optimisation strategies for connecting to my database. It mentioned “pooling” and gave me a sample code which I modified and implemented. I will show you that in a bit.

**What is pooling?**

Database pooling means keeping a set of open database connections in memory so your application can reuse them instead of creating a new connection every time.

At first glance, I see where my error was. For each request, I am making a new database connection. For 10,000 requests, I was making 10,000 connections to the database. That was a good catch. I expected to make new gains.

Updated DB Connection code using pooling:

```go
// database/db.go
```

There were no changes in the health endpoint code.

We will go with the single threaded 10,000 requests test:

```bash
# command for testing

ab -n 10000 http://localhost:8000/health > results.txt
```

Our results:

```plaintext
# results.txt

This is ApacheBench, Version 2.3 <$Revision: 1923142 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient)


Server Software:        
Server Hostname:        localhost
Server Port:            8000

Document Path:          /health
Document Length:        40 bytes

Concurrency Level:      1
Time taken for tests:   545.499 seconds
Complete requests:      10000
Failed requests:        0
Total transferred:      1620000 bytes
HTML transferred:       400000 bytes
Requests per second:    18.33 [#/sec] (mean)
Time per request:       54.550 [ms] (mean)
Time per request:       54.550 [ms] (mean, across all concurrent requests)
Transfer rate:          2.90 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        2    6   3.1      5      90
Processing:    29   48  18.4     44     449
Waiting:       27   47  17.9     43     440
Total:         32   54  20.1     50     470

Percentage of the requests served within a certain time (ms)
  50%     50
  66%     53
  75%     56
  80%     58
  90%     66
  95%     82
  98%    107
  99%    136
 100%    470 (longest request)
```

We see something interesting happening here. The API went for **a 100% success rate at the cost of speed**. Yeah, every request was successful and the database did not complain of an overload, but **545.5 seconds** for 10,000 requests is just too much. We had only **18 requests processed each second** with **54.55 ms spent on each request**. I don't know if I should call this a win but let's see if we make the requests concurrently, there will be any difference.

```bash
# command for testing

ab -n 10000 -c 100 http://localhost:8000/health > results.txt
```

```plaintext
# results.txt

This is ApacheBench, Version 2.3 <$Revision: 1923142 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient)


Server Software:        
Server Hostname:        localhost
Server Port:            8000

Document Path:          /health
Document Length:        40 bytes

Concurrency Level:      100
Time taken for tests:   494.513 seconds
Complete requests:      10000
Failed requests:        643
   (Connect: 0, Receive: 0, Length: 643, Exceptions: 0)
Non-2xx responses:      643
Total transferred:      1624501 bytes
HTML transferred:       393570 bytes
Requests per second:    20.22 [#/sec] (mean)
Time per request:       4945.125 [ms] (mean)
Time per request:       49.451 [ms] (mean, across all concurrent requests)
Transfer rate:          3.21 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        1    9  14.8      6     206
Processing:   121 4444 6988.8   3085  120038
Waiting:       33 4442 6988.6   3083  120034
Total:        122 4453 6992.9   3094  120212

Percentage of the requests served within a certain time (ms)
  50%   3094
  66%   3953
  75%   4701
  80%   5132
  90%   7034
  95%   9569
  98%  15187
  99%  23437
 100%  120212 (longest request)
```

Well, well, well. We have 643 failed requests (6.43%). The time taken to complete the 10,000 requests is still too much: 494.5 seconds. 49.45 ms is spent on each request with 20 requests processed per second.

After reading my code over and over, I realized that I was doing something wrong. I was creating multiple pools. I wasn't doing anything different from the original standard method. However, it was overwhelmed the database less.

How do we get the best of both worlds?

## Test #3: Pooling Done Right

The concept of pooling obviously was the solution, but I faltered in the implementation. What now happens is that, I create multiple pools with limited connections as opposed to multiple connections with no pooling. I figured: what if I just need one pool? Then I asked ChatGPT what that looks like. It gave me a sample code. Again, I had to make adjustments for my codebase because I didn't share it with the AI. With this implementation, only one pool is created for the API to use and that pool has a connection cap that does not overwhelm the database. Let's checkout the test results for this:

```go
// main.go
func main() {
	ctx := context.Background()

    // Create only one pool for the API to use
	pool, err := database.NewPool(ctx)
	if err != nil {
		log.Fatal(err)
	}
	defer pool.Close()

    // As you can see, I pass the pool through the layers of the API
	service := services.NewService(pool)
	handler := handlers.NewHandler(service)
	router := internalHTTP.NewRouter(handler)

	log.Println("API server started on http://localhost:8000")
	if os.Getenv("ENVIRONMENT") == "dev" {
		log.Println("API docs available at http://localhost:8000/swagger/index.html")
		log.Println("API profiler available at http://localhost:8000/debug")
	}

	server := &http.Server{
		Addr:    ":8000",
		Handler: router,
	}

	if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
		log.Fatal("Server error:", err)
	}

	// Graceful shutdown
	stop := make(chan os.Signal, 1)
	signal.Notify(stop, os.Interrupt, syscall.SIGTERM)

	<-stop
	log.Println("Shutting down server...")

	ctxShutdown, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

	server.Shutdown(ctxShutdown)
	log.Println("Server stopped gracefully")
}
```

```go
// internal/database/db.go
```

```go
// internal/http/handlers/health.go
func (h *Handler) HealthCheck(w http.ResponseWriter, r *http.Request) *types.ErrorDetails {
	ctx := context.Background()
	// check if loading configurations are successful
	_, err := config.LoadDBConfig()
	if err != nil {
		types.ReturnJSON(w, types.ServiceUnavailableErrorResponse())
		return &types.ErrorDetails{
			Trace:   err,
			Message: "Failed to load database config",
		}
	}

	// check if jwt config is loaded successfully
	_, err = config.LoadJWTConfig()
	if err != nil {
		types.ReturnJSON(w, types.ServiceUnavailableErrorResponse())
		return &types.ErrorDetails{
			Trace:   err,
			Message: "Failed to load jwt config",
		}
	}

	// check db ping
	if err = h.Service.DB.Ping(ctx); err != nil {
		types.ReturnJSON(w, types.ServiceUnavailableErrorResponse())
		return &types.ErrorDetails{
			Trace:   err,
			Message: "Failed to ping DB",
		}
	}

	return types.ReturnJSON(w, types.OKResponse("Service is up", nil))
}
```

Same test for 10,000 requests from one user:

```bash
# command for testing

ab -n 10000 http://localhost:8000/health > results.txt
```

```plaintext
# results.txt

This is ApacheBench, Version 2.3 <$Revision: 1923142 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient)


Server Software:        
Server Hostname:        localhost
Server Port:            8000

Document Path:          /health
Document Length:        40 bytes

Concurrency Level:      1
Time taken for tests:   38.947 seconds
Complete requests:      10000
Failed requests:        0
Total transferred:      1620000 bytes
HTML transferred:       400000 bytes
Requests per second:    256.76 [#/sec] (mean)
Time per request:       3.895 [ms] (mean)
Time per request:       3.895 [ms] (mean, across all concurrent requests)
Transfer rate:          40.62 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        1    2   0.7      2      11
Processing:     1    2   0.7      2      10
Waiting:        0    1   0.5      1       9
Total:          2    4   1.1      3      15

Percentage of the requests served within a certain time (ms)
  50%      3
  66%      4
  75%      4
  80%      4
  90%      5
  95%      6
  98%      7
  99%      8
 100%     15 (longest request)
```

Now, we are talking. Yeah, I did scream the first time this run. That joy when things work and you can tell why. That spark was what made us study computer science and problem solving. Now, we have a **100% success rate** with **257 requests processed per second**. That is **14.28x more requests** than our previous implementation. It took **38.95 seconds** to complete 10,000 requests with 3.9ms spent on each request. That is **14x faster** than our previous implementation. This is significant gains.

I had high expectations for concurrent users and I wasn't disappointed:

```bash
# command for testing

ab -n 10000 -c 100 http://localhost:8000/health > results.txt
```

```plaintext
This is ApacheBench, Version 2.3 <$Revision: 1923142 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking localhost (be patient)


Server Software:        
Server Hostname:        localhost
Server Port:            8000

Document Path:          /health
Document Length:        40 bytes

Concurrency Level:      100
Time taken for tests:   21.398 seconds
Complete requests:      10000
Failed requests:        0
Total transferred:      1620000 bytes
HTML transferred:       400000 bytes
Requests per second:    467.32 [#/sec] (mean)
Time per request:       213.984 [ms] (mean)
Time per request:       2.140 [ms] (mean, across all concurrent requests)
Transfer rate:          73.93 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        1  104  17.9     99     196
Processing:    33  109  24.5    101     292
Waiting:        2   72  30.3     72     192
Total:         83  213  37.6    197     465

Percentage of the requests served within a certain time (ms)
  50%    197
  66%    220
  75%    234
  80%    241
  90%    264
  95%    295
  98%    322
  99%    327
 100%    465 (longest request)
```

It took **21.4 seconds** to process 10,000 requests from 100 concurrent users with **100% success rate**. The database is not overwhelmed and speed is not compromised. This is what I wanted. We have **467 requests processed in a second with 2.14 ms spent on each request**. I am a happy man.

## Summary of Performance gains

<table><tbody><tr><td colspan="1" rowspan="1"><p></p></td><th colspan="1" rowspan="1"><p><strong>Processed Requests (mean)</strong></p></th><th colspan="1" rowspan="1"><p><strong>Requests per second (mean)</strong></p></th><th colspan="1" rowspan="1"><p><strong>Time per request (mean)</strong></p></th></tr><tr><td colspan="1" rowspan="1"><p><strong>Original Approach (single threaded)</strong></p></td><td colspan="1" rowspan="1"><p>3,700</p></td><td colspan="1" rowspan="1"><p>44</p></td><td colspan="1" rowspan="1"><p>22.595 ms</p></td></tr><tr><td colspan="1" rowspan="1"><p><strong>Final Solution (Single threaded)</strong></p></td><td colspan="1" rowspan="1"><p>10,000</p></td><td colspan="1" rowspan="1"><p>257</p></td><td colspan="1" rowspan="1"><p>3.895 ms</p></td></tr><tr><td colspan="1" rowspan="1"><p><strong>Gain</strong></p></td><td colspan="1" rowspan="1"><p>+270%</p></td><td colspan="1" rowspan="1"><p>+584.09%</p></td><td colspan="1" rowspan="1"><p>+580.1%</p></td></tr></tbody></table>

|  | **Processed Requests (mean)** | **Requests per second (mean)** | **Time per request (mean)** |
| --- | --- | --- | --- |
| **Original Approach (with concurrency)** | 2,900 | 116 | 8.587 ms |
| **Final Solution (with concurrency)** | 10,000 | 467 | 2.14 ms |
| **Gains** | 344.83% | +402.59% | +401.26% |

## Lessons

It is not that AI will kill software engineering. It will change how we arrive at solutions. Back in 2017, what we did was to go on forums and stack overflow to validate any ideas we had and allow seniors to correct us. If luck was not on our side, we go read the manual. Now, all that is consolidated into Generative AI. It makes testing of ideas much faster, but only at the rate of the developer's experience. If I knew nothing about database systems and did not understand the concept of pooling to realize that there was a flaw in the initial implementation, I would have solved the problem, but not efficiently. Don't outsource your critical thinking to an LLM. Rather, explore, learn and enjoy the good feeling of stepping back and looking at what you've built and say: “This is awesome”.

See you in the next episode of documenting curiosity. Take care!

And yeah, this is awesome.