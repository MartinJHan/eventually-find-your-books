# Eventually Find Your Books

A scalable book search and recommendation system built with microservices, DynamoDB, Redis caching, Docker, and AWS infrastructure.

This project was designed to explore how distributed systems behave under realistic search and recommendation workloads. The system supports searching a large book catalog, retrieving book details, storing user ratings, and generating personalized recommendations. We also performed load testing and optimization experiments to improve latency, throughput, reliability, and cost efficiency.

## Team

**Team Readis**

- Mansi Modi
- Junyao Han
- Snahil Dasawat
- Theodore Pei

## Project Overview

Eventually Find Your Books is a backend system for searching and recommending books from a large catalog. The main challenge is that users expect fast search responses even when the catalog contains tens of thousands of books and the system is under concurrent load.

The project focuses on three major engineering questions:

1. Can horizontal scaling alone reduce search latency?
2. How much can sharding improve search performance?
3. How much does Redis caching improve recommendation and search workloads?

Through testing, we found that simply adding more application workers did not significantly improve latency because the real bottleneck was database access. Better results came from optimizing the data access pattern through balanced sharding and Redis caching.

## Features

- Book search API
- Book detail API
- User rating submission
- Rating lookup by user
- Personalized recommendation API
- Redis-based caching
- DynamoDB-backed storage
- Dockerized local development
- AWS-ready infrastructure
- Load testing with Locust
- Performance comparison across scaling, sharding, and caching strategies

## Tech Stack

- **Backend:** Python FastAPI, Go
- **Database:** Amazon DynamoDB
- **Cache:** Redis
- **Containerization:** Docker, Docker Compose
- **Infrastructure:** Terraform, AWS ECS / Load Balancer
- **Testing:** Postman, Locust
- **Deployment:** AWS cloud infrastructure

## Repository Structure

```text
eventually-find-your-books/
├── app/                     # Application-level code
├── data-processing/          # Data preparation and processing scripts
├── infrastructure/           # Terraform / cloud infrastructure files
├── scripts/                  # Utility and deployment scripts
├── services/                 # Backend microservices
├── Dockerfile                # Main Docker build file
├── docker-compose.yml        # Local multi-service setup
├── DOCKER_README.md          # Docker-specific setup guide
├── requirements.txt          # Python dependencies
└── README.md
```

## Main Services

### Search API

The Search API handles book search requests over the book catalog. Early testing showed that direct DynamoDB scan operations caused high tail latency, especially at the 95th percentile. To improve this, the project explored sharded search workers and balanced composite sharding.

### Book Detail API

The Book Detail API retrieves information for a specific book ID. Compared with search, this API is faster because it uses direct item lookups instead of full-table scans.

### Recommendation API

The Recommendation API computes personalized book recommendations based on user ratings. It uses collaborative filtering logic and Redis caching to reduce repeated database reads and improve response time.

My main contribution was the Recommendation API. I implemented a Python FastAPI microservice that retrieves user rating data, computes personalized recommendations, integrates Redis caching, and works as part of the larger distributed system.

## API Examples

### Search Books

```bash
curl -X POST http://localhost:8080/search \
  -H "Content-Type: application/json" \
  -d '{"query":"Ready Player One","limit":5}'
```

### Get Book Details

```bash
curl http://localhost:8081/books/OL15936512W
```

### Add a Rating

```bash
curl -X POST http://localhost:8080/books/OL15936512W/rate \
  -H "Content-Type: application/json" \
  -d '{"user_id":"alice","rating":5}'
```

### Get User Ratings

```bash
curl http://localhost:8080/users/alice/ratings
```

### Get Recommendations

```bash
curl http://localhost:8000/recommendations/alice
```

## Running Locally with Docker

Make sure Docker and Docker Compose are installed.

Build and start the services:

```bash
docker compose build
docker compose up -d
```

Check running containers:

```bash
docker compose ps
```

Check service health:

```bash
curl http://localhost:8080/healthz
curl http://localhost:8081/healthz
```

Check Redis:

```bash
docker exec -it redis redis-cli ping
```

Expected response:

```text
PONG
```

Stop the system:

```bash
docker compose down
```

Remove containers and volumes:

```bash
docker compose down -v
```

## Testing

The project was tested using Postman for API correctness and Locust for load testing.

Initial API testing verified:

- Search requests returned matching books.
- Book detail requests returned the correct metadata.
- Rating submission successfully stored user ratings.
- User rating lookup returned the stored rating.
- Recommendation requests returned personalized book IDs.

Initial load testing without caching and sharding showed:

| Metric | Result | Target | Status |
|---|---:|---:|---|
| Failures | 0% | < 1% | Excellent |
| Throughput | 70 RPS | 50+ RPS | Good |
| P50 Latency | ~100 ms | < 200 ms | Excellent |
| P95 Latency | ~1700 ms | < 500 ms | Needs Improvement |
| Stability | Steady | No crashes | Excellent |

The main bottleneck was DynamoDB scan operations in the Search API. Search requests were much slower than direct book detail lookups because scans required reading across many items.

## Performance Experiments

### 1. Horizontal Scaling

We tested whether adding more application tasks would improve latency.

Setup:

- 500 concurrent users
- 15-minute sustained load
- 2, 5, and 10 search workers

| Tasks | P95 Latency | Throughput | Failures |
|---:|---:|---:|---:|
| 2 | 160 ms | 231 RPS | 0% |
| 5 | 160 ms | 231 RPS | 0% |
| 10 | 150 ms | 231 RPS | 0% |

Adding more workers only reduced P95 latency by about 6%. This showed that the application layer was not the main bottleneck. The database access pattern was the limiting factor.

### 2. Sharding Strategy

We tested three search strategies:

| Strategy | Description | P95 Latency | Result |
|---|---|---:|---|
| Sequential Search | 1 worker, full-table scan | 3800 ms | Baseline |
| Naive A-Z Sharding | 26 workers by first letter | 180 ms | 21x speedup |
| Balanced Composite Sharding | 16 balanced workers | 150 ms | 25x speedup |

The best result came from balanced composite sharding. Although A-Z sharding used more workers, it created uneven workloads because some letters had many more books than others. For example, titles starting with “T” created a hotspot. Balanced sharding distributed the workload more evenly and reduced straggler effects.

### 3. Redis Cache

We tested Redis caching under a 250-user, 20-minute workload.

| Configuration | Throughput | P95 Latency | Failures |
|---|---:|---:|---:|
| No Cache | 130 RPS | 53,000 ms | 9% |
| 1-min TTL | 41 RPS | 60,000 ms | 5.5% |
| 10-min TTL | 651 RPS | 730 ms | 0.16% |

The 10-minute TTL cache provided the best performance. It increased throughput by about 5x and reduced failure rate from 9% to 0.16%.

The 1-minute TTL performed worse than no cache because frequent invalidation caused cache thrashing and a thundering herd effect.

## Final Results

After applying the main optimizations, the system improved significantly:

| Metric | Baseline | Final Optimized System |
|---|---:|---:|
| Throughput | 130 RPS | 651 RPS |
| Average Latency | 1842 ms | 353 ms |
| Failure Rate | 9% | 0.16% |
| CPU Usage | 100% | 75% |
| DynamoDB Cost | Baseline | ~94% reduction |

## Key Engineering Lessons

### Measure Before Scaling

Adding more servers was not the real solution. Load testing showed that the bottleneck was database access, not application compute.

### Balance Matters More Than Quantity

Using 16 balanced shards performed better than 26 unbalanced alphabetical shards. A smaller number of well-balanced workers can outperform a larger number of uneven workers.

### Caching Requires the Right TTL

Redis improved performance only when the cache TTL matched the workload pattern. A poorly chosen TTL created more overhead than benefit.

### Production Systems Need Safe Rollbacks

This project also highlighted the difference between classroom deployments and production systems. In production, data cannot simply be reset. We considered blue-green deployments, feature flags, rollback strategies, and safer migration plans for sharding and cache configuration changes.

## Future Improvements

- Add authentication for user-specific recommendation endpoints
- Improve recommendation quality with more advanced collaborative filtering
- Add pagination and ranking to search results
- Add observability dashboards for latency, throughput, cache hit rate, and DynamoDB usage
- Add CI/CD pipeline for automated testing and deployment
- Improve infrastructure automation and deployment safety

## Summary

Eventually Find Your Books is a distributed book search and recommendation system built to study real backend performance problems. The project showed that the best optimization is not always adding more servers. The largest gains came from understanding the bottleneck, improving the database access pattern with balanced sharding, and applying Redis caching with a workload-appropriate TTL.
