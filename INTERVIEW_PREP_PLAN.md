# Interview Preparation Plan

## Overview

This document outlines a comprehensive interview preparation structure covering 5 roles:
- JavaScript Individual Contributor
- System Design
- AWS
- Python
- Fullstack

---

## Proposed Folder Structure

```
Interview-Prep/
├── 00-Common/                    # Shared across all roles
│   ├── 01-DSA/                   # Data Structures & Algorithms
│   ├── 02-Behavioral/            # STAR method, Leadership principles
│   └── 03-Problem-Solving/       # General coding patterns
│
├── 01-JavaScript-IC/
│   ├── 01-Core-Concepts/
│   ├── 02-Advanced-Topics/
│   ├── 03-Coding-Challenges/
│   └── 04-Best-Practices/
│
├── 02-System-Design/
│   ├── 01-Fundamentals/
│   ├── 02-Components/
│   ├── 03-Case-Studies/
│   └── 04-Practice-Problems/
│
├── 03-AWS/
│   ├── 01-Core-Services/
│   ├── 02-Architecture-Patterns/
│   ├── 03-Security-Best-Practices/
│   └── 04-Scenario-Questions/
│
├── 04-Python/
│   ├── 01-Core-Concepts/
│   ├── 02-Advanced-Topics/
│   ├── 03-Coding-Challenges/
│   └── 04-Libraries-Frameworks/
│
└── 05-Fullstack/
    ├── 01-Frontend/
    ├── 02-Backend/
    ├── 03-Databases/
    └── 04-DevOps/
```

---

## Detailed Content Plan

---

### 00-Common (Shared Topics)

#### 01-DSA (Data Structures & Algorithms)

| Topic | Content |
|-------|---------|
| Arrays & Strings | Two pointers, sliding window, prefix sums |
| Linked Lists | Reversal, cycle detection, merge operations |
| Stacks & Queues | Monotonic stack, BFS applications |
| Hash Tables | Collision handling, frequency counting |
| Trees & BST | Traversals, LCA, balanced trees |
| Graphs | BFS, DFS, Dijkstra, topological sort |
| Dynamic Programming | Memoization, tabulation, common patterns |
| Recursion & Backtracking | Permutations, combinations, N-Queens |
| Sorting & Searching | QuickSort, MergeSort, Binary Search variants |
| Heaps & Priority Queues | Top-K problems, merge K sorted lists |

#### 02-Behavioral

| Topic | Content |
|-------|---------|
| STAR Method | Situation, Task, Action, Result framework |
| Amazon Leadership Principles | All 16 principles with example answers |
| Common Questions | Conflict, failure, leadership, teamwork stories |
| Company Research | How to research company culture & values |

#### 03-Problem-Solving

| Topic | Content |
|-------|---------|
| Coding Patterns | 15 essential patterns (sliding window, fast/slow pointers, etc.) |
| Time/Space Complexity | Big-O analysis cheat sheet |
| Communication | How to think aloud, ask clarifying questions |

---

### 01-JavaScript-IC

#### 01-Core-Concepts

| Topic | Content |
|-------|---------|
| Execution Context | Call stack, scope chain, hoisting |
| Closures | Lexical scope, practical use cases |
| this Keyword | Binding rules, arrow functions |
| Prototypes | Prototype chain, inheritance |
| Event Loop | Call stack, task queue, microtasks |
| Promises | Creation, chaining, error handling |
| Async/Await | Syntax, error handling, parallelism |
| ES6+ Features | Destructuring, spread, modules, classes |

#### 02-Advanced-Topics

| Topic | Content |
|-------|---------|
| Memory Management | Garbage collection, memory leaks |
| Performance | Debouncing, throttling, lazy loading |
| Design Patterns | Singleton, Observer, Factory, Module |
| TypeScript | Types, generics, utility types, type guards |
| Web APIs | DOM, Fetch, Storage, Web Workers |
| Security | XSS, CSRF, CSP, sanitization |

#### 03-Coding-Challenges

| Topic | Content |
|-------|---------|
| Implement Polyfills | Promise, bind, map, reduce, debounce |
| DOM Manipulation | Event delegation, virtual DOM concepts |
| Async Challenges | Promise.all, race conditions, retry logic |

#### 04-Best-Practices

| Topic | Content |
|-------|---------|
| Code Quality | Clean code, SOLID principles |
| Testing | Jest, unit testing, mocking |
| Module Systems | CommonJS vs ES Modules |

---

### 02-System-Design

#### 01-Fundamentals

| Topic | Content |
|-------|---------|
| Scalability | Horizontal vs vertical scaling |
| CAP Theorem | Consistency, Availability, Partition tolerance |
| Load Balancing | Algorithms, Layer 4 vs Layer 7 |
| Caching | Cache invalidation, CDN, Redis |
| Database Sharding | Partitioning strategies |
| Replication | Master-slave, master-master |
| Consistency Models | Strong, eventual, causal |

#### 02-Components

| Topic | Content |
|-------|---------|
| Databases | SQL vs NoSQL, when to use what |
| Message Queues | Kafka, RabbitMQ, use cases |
| API Design | REST vs GraphQL, versioning |
| Microservices | Service discovery, communication |
| Rate Limiting | Token bucket, sliding window |
| Search Systems | Elasticsearch, inverted index |

#### 03-Case-Studies

| Topic | Content |
|-------|---------|
| URL Shortener | TinyURL design |
| Social Media Feed | Twitter/Instagram feed |
| Chat System | WhatsApp/Slack design |
| Video Streaming | YouTube/Netflix design |
| E-commerce | Amazon-like system |
| Ride Sharing | Uber/Lyft design |
| File Storage | Dropbox/Google Drive |
| Notification System | Push notifications at scale |

#### 04-Practice-Problems

| Topic | Content |
|-------|---------|
| RESHADED Framework | Requirements, Estimation, Storage, High-level, API, Detailed, Evaluation, Deployment |
| Trade-off Analysis | How to discuss pros/cons |
| Back-of-envelope Calculations | QPS, storage, bandwidth estimation |

---

### 03-AWS

#### 01-Core-Services

| Topic | Content |
|-------|---------|
| Compute | EC2, Lambda, ECS, EKS, Fargate |
| Storage | S3, EBS, EFS, Glacier |
| Database | RDS, DynamoDB, Aurora, ElastiCache, Redshift |
| Networking | VPC, Route 53, CloudFront, API Gateway |
| Messaging | SQS, SNS, EventBridge, Kinesis |

#### 02-Architecture-Patterns

| Topic | Content |
|-------|---------|
| Well-Architected Framework | 6 pillars deep dive |
| High Availability | Multi-AZ, multi-region |
| Serverless | Lambda patterns, Step Functions |
| Microservices on AWS | ECS/EKS patterns |
| Event-Driven | EventBridge, SNS/SQS patterns |

#### 03-Security-Best-Practices

| Topic | Content |
|-------|---------|
| IAM | Roles, policies, least privilege |
| Encryption | KMS, at-rest, in-transit |
| Network Security | Security groups, NACLs, WAF |
| Secrets Management | Secrets Manager, Parameter Store |
| Compliance | CloudTrail, Config, GuardDuty |

#### 04-Scenario-Questions

| Topic | Content |
|-------|---------|
| Disaster Recovery | Backup strategies, RTO/RPO |
| Cost Optimization | Reserved instances, spot, rightsizing |
| Migration | 6 R's, migration strategies |
| Troubleshooting | Common issues and solutions |

---

### 04-Python

#### 01-Core-Concepts

| Topic | Content |
|-------|---------|
| Data Types | Mutable vs immutable, type hints |
| OOP | Classes, inheritance, magic methods |
| Iterators & Generators | yield, generator expressions |
| Decorators | Function & class decorators |
| Context Managers | with statement, custom managers |
| Exception Handling | try/except, custom exceptions |

#### 02-Advanced-Topics

| Topic | Content |
|-------|---------|
| Memory Management | GIL, garbage collection, reference counting |
| Metaclasses | Class creation, customization |
| Concurrency | Threading, multiprocessing, asyncio |
| Descriptors | Property, __get__, __set__ |
| Closures & Scoping | LEGB rule, nonlocal |

#### 03-Coding-Challenges

| Topic | Content |
|-------|---------|
| Pythonic Solutions | List comprehensions, one-liners |
| Common Algorithms | Implemented in Python |
| String Manipulation | Regex, parsing |

#### 04-Libraries-Frameworks

| Topic | Content |
|-------|---------|
| Django/Flask/FastAPI | Web frameworks comparison |
| Pandas/NumPy | Data manipulation |
| Testing | pytest, unittest, mocking |
| Type Checking | mypy, Pydantic |

---

### 05-Fullstack

#### 01-Frontend

| Topic | Content |
|-------|---------|
| React Fundamentals | Components, props, state, lifecycle |
| React Hooks | useState, useEffect, useContext, custom hooks |
| State Management | Redux, Context API, Zustand |
| Performance | Memoization, code splitting, lazy loading |
| HTML/CSS | Semantic HTML, Flexbox, Grid, accessibility |

#### 02-Backend

| Topic | Content |
|-------|---------|
| Node.js/Express | Middleware, routing, error handling |
| REST API Design | HTTP methods, status codes, versioning |
| GraphQL | Queries, mutations, resolvers |
| Authentication | JWT, OAuth, session management |
| Validation | Input validation, sanitization |

#### 03-Databases

| Topic | Content |
|-------|---------|
| SQL | Joins, indexing, query optimization |
| NoSQL | MongoDB, document design |
| ORMs | Sequelize, Prisma, SQLAlchemy |
| Transactions | ACID, isolation levels |

#### 04-DevOps

| Topic | Content |
|-------|---------|
| Docker | Containers, Dockerfile, compose |
| CI/CD | GitHub Actions, Jenkins pipelines |
| Git | Branching strategies, rebasing, conflicts |
| Monitoring | Logging, metrics, alerting |

---

## File Types Per Topic

For each topic, we'll create:

| File | Purpose |
|------|---------|
| `Concepts.md` | Theory and explanations |
| `Questions.md` | Interview questions with answers |
| `Cheatsheet.md` | Quick reference cards |
| `Practice.md` | Hands-on exercises and challenges |
| `Resources.md` | Links to documentation, courses, videos |

---

## Research Sources

### JavaScript
- [GreatFrontEnd - Top JavaScript Interview Questions](https://github.com/greatfrontend/top-javascript-interview-questions)
- [EngX Space - Senior JavaScript Questions](https://engx.space/global/en/blog/senior-javascript-developer-interview-questions)
- [InterviewBit - JavaScript Questions](https://www.interviewbit.com/javascript-interview-questions/)
- [Toptal - JavaScript Interview Questions](https://www.toptal.com/javascript/interview-questions)

### System Design
- [Design Gurus - System Design Interview Guide 2025](https://www.designgurus.io/blog/system-design-interview-guide-2025)
- [Interviewing.io - Senior Engineer's Guide](https://interviewing.io/guides/system-design-interview)
- [Hello Interview - System Design in a Hurry](https://www.hellointerview.com/learn/system-design/in-a-hurry/introduction)
- [GeeksforGeeks - System Design at FAANG](https://www.geeksforgeeks.org/system-design/system-design-interviews-faang/)

### AWS
- [K21 Academy - AWS Solutions Architect Questions](https://k21academy.com/amazon-web-services/aws-solutions-architect/interview-questions-and-answers/)
- [Edureka - Top 150+ AWS Questions](https://www.edureka.co/blog/interview-questions/aws-interview-questions/)
- [Digital Cloud Training - AWS Interview Questions](https://digitalcloud.training/top-20-aws-solutions-architect-interview-questions/)

### Python
- [GeeksforGeeks - Top 50+ Python Questions](https://www.geeksforgeeks.org/python/python-interview-questions/)
- [Toptal - Senior Python Developer Questions](https://www.toptal.com/external-blogs/youteam/senior-python-developer-interview-questions-and-answers)
- [GitHub - Python Questions for Senior/Lead](https://github.com/matacoder/senior)

### Fullstack
- [GeeksforGeeks - 200+ Fullstack Questions](https://www.geeksforgeeks.org/html/full-stack-developer-interview-questions-and-answers/)
- [InterviewBit - Fullstack Developer Questions](https://www.interviewbit.com/full-stack-developer-interview-questions/)
- [Frontend Interview Handbook](https://www.frontendinterviewhandbook.com/introduction)

### DSA & Behavioral
- [Tech Interview Handbook - DSA Cheatsheets](https://www.techinterviewhandbook.org/algorithms/study-cheatsheet/)
- [Design Gurus - FAANG Behavioral Guide](https://www.designgurus.io/blog/faang-behavioral-interview-guide)
- [MIT - STAR Method Guide](https://capd.mit.edu/resources/the-star-method-for-behavioral-interviews/)

---

## Questions Before Implementation

1. **Do you want all these topics, or should we add/remove any?**

2. **Any specific companies you're targeting?** (Google, Amazon, Meta, startups, etc.)

3. **What's your experience level?** (This helps calibrate difficulty)

4. **Any specific technologies within Fullstack?** (e.g., React vs Vue, Node vs Django)

5. **Do you want code examples in the files, or just concepts and questions?**

---

## Status

- [ ] Plan approved
- [ ] Folder structure created
- [ ] Common topics completed
- [ ] JavaScript IC completed
- [ ] System Design completed
- [ ] AWS completed
- [ ] Python completed
- [ ] Fullstack completed

---

*Last Updated: February 2025*


<!-- event loop for nodejs as well -->