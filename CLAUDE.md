
## Workflow
- always start with a plan
- do not change anything without acceptance. Before writing any code, describe your approach and wait for approval. Always ask clarifying questions before writing any code if requirements are ambiguous
- try to add changes in small steps which can be seperate commits
- after each step, run tests, build, and lint before presenting the step as ready to commit
- every time I correct you, suggest a new rule to the CLAUDE .md file so it never happens again


## Key Principles
1. Simplicity and readability over cleverness
2. Explicit is better than implicit
3. Fail fast with clear error messages
4. Handle all errors explicitly (Go) / use exceptions appropriately (Python)
5. Test-driven development. Write tests before implementation. Do not test private methods. Split test functions into separate Allowed/NotAllowed (or Success/Failure) tests instead of using an `expected` field in table-driven tests
6. Consistent code formatting and style
7. Comprehensive documentation for public APIs and complex logic
8. Modular design to promote reusability
9. Use version control effectively with meaningful commit messages
10. Domain-Driven Design (DDD) principles where applicable
11. Application logic separated from transport and infrastructure concerns. Use interfaces instead of direct implementation.
12. Use dependency injection for better testability
13. Observability: logging, metrics, tracing
14. Integration tests using testcontainers or similar tools
15. For API use openapi/Swagger for documentation and client generation (and server is possible)
