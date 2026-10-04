# Schema Labs — Research


### Notes: Oct 4, 2026 - Problem Statement

I started Schema Labs with a small question: why is changing a database schema while an application is running still so difficult?

Take `orders.status`. It’s a text column that holds just four values: `pending`, `paid`, `shipped`, and `cancelled`. I want to store those as small integers instead. It sounds simple, but I want the database to stay live, and I don’t want every application using it to need a code change at the same time.

You could run an `ALTER` and deal with the disruption, work through the change in stages, or use a tool that supports online migrations. But knowing that a tool can migrate the data doesn’t quite answer what I’d want to know before using it: will my app keep working while this happens?

That’s what I’m trying to understand here. I have identified some approaches and databases for which I will read their papers, try them and see what happens with the queries and drivers applications actually use. I initially thought of making my own diary but I think this will be a much more interesting thing to do- write down what I learnt, what works and what breaks

This repo holds will hopefully hold those research & notes. I have created another [Benchmark](https://github.com/DBSchemaLabs/Benchmarks) harness repo which is more of the benchmark I want to run against these approaches.